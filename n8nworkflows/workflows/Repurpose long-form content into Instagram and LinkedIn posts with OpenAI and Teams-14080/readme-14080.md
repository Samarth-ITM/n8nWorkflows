Repurpose long-form content into Instagram and LinkedIn posts with OpenAI and Teams

https://n8nworkflows.xyz/workflows/repurpose-long-form-content-into-instagram-and-linkedin-posts-with-openai-and-teams-14080


# Repurpose long-form content into Instagram and LinkedIn posts with OpenAI and Teams

# 1. Workflow Overview

This workflow turns a long-form content attachment received by Gmail into multiple social media assets, using OpenAI for content analysis and generation, Microsoft Teams for human approval, Google Drive and Google Sheets for asset management and tracking, APITemplate.io / HTML-to-PDF for visuals, and Blotato for publishing.

Primary use cases:
- Repurposing podcast transcripts, newsletters, scripts, or other long-form documents into short-form and LinkedIn assets
- Keeping a documented asset trail in Google Drive and Google Sheets
- Adding human-in-the-loop review before publishing

The workflow has one technical entry point: **Gmail Trigger**. From there it branches into four major production paths after a shared analysis stage:
1. Instagram / YouTube-style social carousel
2. LinkedIn carousel
3. LinkedIn text-only post
4. LinkedIn media post

## 1.1 Input Reception and Source Archiving
The workflow watches Gmail for messages matching a search query, downloads the attachment, extracts text, stores the original content in Google Drive, creates a project subfolder, extracts any URL from the email body, and logs run metadata in Google Sheets.

## 1.2 Strategic AI Content Extraction
A core OpenAI agent analyzes the extracted text and produces five structured content pillars. A validation loop ensures that all five topics exist and are non-empty before downstream branches continue.

## 1.3 Instagram / Social Carousel Branch
Using the content pillars, an AI specialist generates a social caption plus five short slide texts. A Teams approval loop allows revisions. Once approved, slide images are generated with APITemplate.io, stored in Drive, and published through Blotato. Publication status is polled and reported.

## 1.4 LinkedIn Carousel Branch
Another AI specialist creates a LinkedIn caption, five 2-word hooks, and five slide texts. After Teams review, an HTML-based carousel is rendered to PDF, stored in Drive, and its assets are announced in Teams.

## 1.5 LinkedIn Text Post Branch
A dedicated AI specialist writes a standalone LinkedIn post. After review and approval, the text is saved to Drive, published via Blotato, and its public URL is logged in Google Sheets.

## 1.6 LinkedIn Media Post Branch
A final AI specialist creates a LinkedIn post plus a short guiding statement for a generated image. After approval, an image is created with APITemplate.io, stored in Drive, published with the post via Blotato, and tracked back into Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Source Archiving

**Overview:**  
This block ingests the source email, retrieves the attached file, extracts text from it, stores the original text in Drive, organizes it into a project folder, extracts any source URL from the email body, and logs run metadata in a Google Sheet.

**Nodes Involved:**  
- Gmail Trigger  
- Download Email Attachments  
- Extract from File  
- Save original content to Drive  
- Create project subfolder  
- Move original to project folder  
- Extract URLs from approval data  
- Log run metadata in Google Sheets

### Gmail Trigger
- **Type and role:** `n8n-nodes-base.gmailTrigger`; polling trigger for incoming Gmail messages.
- **Configuration:**  
  - Polls every hour  
  - Uses Gmail search query: `Content Repurposing`
  - Downloads no attachments at trigger stage
- **Key expressions / variables:** none beyond trigger output
- **Connections:**  
  - Output to **Download Email Attachments**
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:**  
  - Gmail OAuth failure  
  - Search query too broad or too narrow  
  - Trigger may pick forwarded emails where subject/body structure varies
- **Sub-workflow reference:** none

### Download Email Attachments
- **Type and role:** `n8n-nodes-base.gmail`; fetches full Gmail message including binary attachments.
- **Configuration:**  
  - Operation: `get`
  - `messageId` from current Gmail item
  - `downloadAttachments: true`
- **Key expressions / variables:**  
  - `={{ $json.id }}`
- **Connections:**  
  - Input from **Gmail Trigger**
  - Output to **Extract from File**
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases / failures:**  
  - No attachment present  
  - Attachment indexed differently than expected  
  - Gmail permissions insufficient

### Extract from File
- **Type and role:** `n8n-nodes-base.extractFromFile`; extracts text from the first binary attachment.
- **Configuration:**  
  - Operation: `text`
  - Binary property: `attachment_0`
  - Destination key: `content_text`
- **Key expressions / variables:** none
- **Connections:**  
  - Input from **Download Email Attachments**
  - Output to **Save original content to Drive**
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases / failures:**  
  - Attachment missing or not named `attachment_0`
  - Unsupported file type
  - Poor text extraction from scanned PDFs or complex formats

### Save original content to Drive
- **Type and role:** `n8n-nodes-base.googleDrive`; creates a text file in Drive containing extracted source text.
- **Configuration:**  
  - Operation: `createFromText`
  - Writes `content_text`
  - Initially saves in Drive root
  - File name includes email subject and date
- **Key expressions / variables:**  
  - Name: `Original_Content_<subject>_<date>`
  - Content: `={{ $json.content_text }}`
- **Connections:**  
  - Input from **Extract from File**
  - Output to **Create project subfolder**
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**  
  - Google Drive auth issues  
  - Invalid filename characters from email subject  
  - Very large extracted text

### Create project subfolder
- **Type and role:** `n8n-nodes-base.googleDrive`; creates a run-specific subfolder.
- **Configuration:**  
  - Resource: `folder`
  - Parent folder is a configured Drive folder ID placeholder
  - Folder name uses email subject and date
- **Key expressions / variables:**  
  - `={{ $('Gmail Trigger').item.json.headers.subject }}_{{ $('Gmail Trigger').item.json.date }}`
- **Connections:**  
  - Input from **Save original content to Drive**
  - Output to **Move original to project folder**
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**  
  - Placeholder folder ID not replaced
  - Duplicate folder names
  - Drive permission problems

### Move original to project folder
- **Type and role:** `n8n-nodes-base.googleDrive`; moves the saved source text file into the project folder.
- **Configuration:**  
  - Operation: `move`
  - File ID from **Save original content to Drive**
  - Destination folder ID from **Create project subfolder**
- **Key expressions / variables:**  
  - File ID: `={{ $('Save original content to Drive').item.json.id }}`
  - Folder ID: `={{ $json.id }}`
- **Connections:**  
  - Input from **Create project subfolder**
  - Output to **Extract URLs from approval data**
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**  
  - Upstream file not created  
  - Folder ID missing  
  - Cross-drive move restrictions

### Extract URLs from approval data
- **Type and role:** `n8n-nodes-base.code`; extracts URLs from the Gmail text body, usually the source podcast/article link.
- **Configuration:**  
  - Iterates through all Gmail Trigger items
  - Uses regex to find `https://...`
  - Returns `extractedUrls`, `firstUrl`, and passes through input item as `driveData`
- **Key expressions / variables:**  
  - Uses `$("Gmail Trigger").all()`
  - Regex: `/(https?:\/\/[^\s]+)/g`
- **Connections:**  
  - Input from **Move original to project folder**
  - Output to **Log run metadata in Google Sheets**
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**  
  - Forwarded email body may contain unrelated URLs  
  - No URL found, yielding `firstUrl = null`
  - Regex may include punctuation in malformed emails

### Log run metadata in Google Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a run-tracking row.
- **Configuration:**  
  - Operation: `append`
  - Writes `Run`, mail URL, mail date, subject, Drive folder URL, original asset URL
  - Uses a configured spreadsheet and `gid=0`
- **Key expressions / variables:**  
  - Run: `={{ Math.floor($now.toSeconds()) }}`
  - Gmail URL built from message ID
  - Drive folder URL built from created folder ID
  - Original asset URL from extracted first URL
- **Connections:**  
  - Input from **Extract URLs from approval data**
  - Output to **The Repurpose Strategist**
- **Version-specific requirements:** typeVersion `4.7`
- **Edge cases / failures:**  
  - Spreadsheet ID placeholder not replaced  
  - Column schema mismatch  
  - Run collisions if two executions start in the same second

---

## 2.2 Strategic AI Content Extraction

**Overview:**  
This block performs the core analysis of the source text. It uses an OpenAI agent with a structured JSON schema and an auto-fixing parser to extract five reusable content pillars. A validation IF node ensures all required topics exist before proceeding.

**Nodes Involved:**  
- The Repurpose Strategist  
- OpenAI Chat Model  
- OpenAI Chat Model (parser)  
- Structured Output Parser  
- Auto-fixing Output Parser  
- Validate content pillars exist  
- Update sheet: content pillars

### OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; main LLM for strategy generation.
- **Configuration:** model `gpt-5.1`
- **Connections:**  
  - AI language model input to **The Repurpose Strategist**
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:** API quota, auth errors, timeouts

### OpenAI Chat Model (parser)
- **Type and role:** LLM used by the auto-fixing parser when output needs repair.
- **Configuration:** model `gpt-5.1`
- **Connections:**  
  - AI language model input to **Auto-fixing Output Parser**
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:** same as above

### Structured Output Parser
- **Type and role:** `outputParserStructured`; enforces a manual JSON schema.
- **Configuration:**  
  - Requires `summary`, `topic1` through `topic5`
  - Each topic is a formatted string block
- **Connections:**  
  - Output parser into **Auto-fixing Output Parser**
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:** model may omit required fields or return invalid JSON

### Auto-fixing Output Parser
- **Type and role:** repairs malformed structured output.
- **Configuration:** default options
- **Connections:**  
  - Receives schema from **Structured Output Parser**
  - Receives parser model from **OpenAI Chat Model (parser)**
  - Feeds parser into **The Repurpose Strategist**
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** cannot fix fundamentally incorrect or incomplete outputs

### The Repurpose Strategist
- **Type and role:** LangChain agent; master content strategist.
- **Configuration:**  
  - Long prompt instructing the agent to identify five narrative content pillars from extracted source text
  - Must return structured JSON matching the schema
  - Uses source text from `$('Extract from File').item.json.content_text`
  - Retries enabled, max 5 tries
- **Key expressions / variables:**  
  - Source text expression from **Extract from File**
- **Connections:**  
  - Input from **Log run metadata in Google Sheets**
  - Output to **Validate content pillars exist**
  - Uses AI model and output parser connections
- **Version-specific requirements:** typeVersion `3.1`
- **Edge cases / failures:**  
  - Long transcripts may hit token limits  
  - Weak extraction if source text is noisy  
  - Prompt/schema mismatch could cause parser repair loops

### Validate content pillars exist
- **Type and role:** `n8n-nodes-base.if`; gatekeeper validating AI output completeness.
- **Configuration:**  
  - Checks `output.topic1` to `output.topic5` each both `exists` and `notEmpty`
- **Connections:**  
  - True output to **Update sheet: content pillars**
  - False output loops back to **The Repurpose Strategist**
- **Version-specific requirements:** typeVersion `2.3`
- **Edge cases / failures:**  
  - Infinite or repeated retry behavior if the model consistently fails  
  - Validation does not assess quality, only presence/non-empty

### Update sheet: content pillars
- **Type and role:** `n8n-nodes-base.googleSheets`; stores the five pillars in the run row.
- **Configuration:**  
  - Operation: `appendOrUpdate`
  - Match on `Run`
  - Writes `topic1`–`topic5`
- **Key expressions / variables:**  
  - Run from **Log run metadata in Google Sheets**
  - Topics from `$json.output.topic1...topic5`
- **Connections:**  
  - Input from **Validate content pillars exist**
  - Output fans out to all four downstream adaptive builders
- **Version-specific requirements:** typeVersion `4.7`
- **Edge cases / failures:**  
  - Sheet schema must include topic columns  
  - If `Run` matching fails, duplicate rows may be created

---

## 2.3 Instagram / Social Carousel Branch

**Overview:**  
This branch creates a social caption and five short slide texts for Instagram/YouTube-style use, sends them for Teams approval, supports revision loops, generates slide images, stores them in Drive, and publishes them through Blotato.

**Nodes Involved:**  
- Adaptive Prompt Builder – IG/YT  
- The IG/YT Specialist  
- OpenAI Chat Model2  
- Simple Memory  
- Structured Output Parser1  
- Auto-fixing Output Parser1  
- OpenAI Chat Model3  
- Review SM carousel in Teams  
- IG/YT Draft Approved?  
- Create SM asset folder  
- Update sheet: SM carousel data  
- Generate SM slide 1  
- Generate SM slide 2  
- Generate SM slide 3  
- Generate SM slide 4  
- Generate SM slide 5  
- Download carousel image 1  
- Download carousel image 2  
- Download carousel image 3  
- Download carousel image 4  
- Download carousel image 5  
- Upload SM image 1 to Drive  
- Upload SM image 2 to Drive  
- Upload SM image 3 to Drive  
- Upload SM image 4 to Drive  
- Upload SM image 5 to Drive  
- Merge all SM carousel images  
- Collect SM image URLs for publishing  
- Save SM post text to Drive  
- Publish SM carousel via Blotato  
- Wait 30s for SM carousel  
- Check SM carousel post status  
- Route SM carousel post status  
- Notify: SM carousel published  
- Notify: SM carousel publish error  
- Update sheet: SM image URLs

### Adaptive Prompt Builder – IG/YT
- **Type and role:** Code node building CREATE vs REPAIR input payload.
- **Configuration:**  
  - Reads source topics from **Update sheet: content pillars**
  - If feedback exists, clears source topics and carries previous generated draft into `output`
  - Looks for previous output from **The IG/YT Specialist**
  - References a feedback node named `Feedback Reference`, which does not exist in this JSON
- **Connections:**  
  - Input from **Update sheet: content pillars**
  - Output to **The IG/YT Specialist**
  - Also receives loopback from **IG/YT Draft Approved?** false path
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**  
  - Broken feedback recovery because referenced node `Feedback Reference` is missing  
  - Revision loop may not capture feedback reliably  
  - Depends on exact node naming

### OpenAI Chat Model2
- **Type and role:** Main LLM for this branch.
- **Configuration:** model `gpt-5.1`
- **Connections:** AI model for **The IG/YT Specialist**

### Simple Memory
- **Type and role:** Buffer memory for the specialist agent.
- **Configuration:**  
  - Session key: current Unix seconds
- **Connections:** AI memory for **The IG/YT Specialist**
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:**  
  - Session key changes each run, so memory is not persistent across revision cycles unless same execution context is preserved

### Structured Output Parser1 / Auto-fixing Output Parser1 / OpenAI Chat Model3
- **Type and role:** Schema enforcement and auto-repair for the IG/YT agent.
- **Configuration:**  
  - Requires `sm_post_text` and `topic1`–`topic5`
- **Connections:**  
  - Structured parser -> auto-fixer -> specialist agent
  - OpenAI Chat Model3 provides repair model
- **Edge cases / failures:** malformed JSON, missing fields

### The IG/YT Specialist
- **Type and role:** Agent generating social one-liners and a caption.
- **Configuration:**  
  - CREATE mode uses source content pillars
  - REPAIR mode uses previous draft + feedback
  - Strict brevity rules: max 25 words per topic, max 40 words for intro/caption
- **Connections:**  
  - Input from **Adaptive Prompt Builder – IG/YT**
  - Output to **Review SM carousel in Teams**
- **Edge cases / failures:**  
  - Output path inconsistencies: later nodes reference both `output.sm_post_text` and `output.parameters.sm_post_text`
  - Repair mode depends on feedback builder correctness

### Review SM carousel in Teams
- **Type and role:** Microsoft Teams send-and-wait review form.
- **Configuration:**  
  - Sends HTML message showing caption and five slides
  - Custom form fields: radio `approved` yes/no, text `Feedback`
- **Connections:**  
  - Input from **The IG/YT Specialist**
  - Output to **IG/YT Draft Approved?**
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**  
  - Teams OAuth/chat ID issues  
  - Long content may render poorly in Teams  
  - Human reviewer may submit empty feedback with “no”

### IG/YT Draft Approved?
- **Type and role:** IF node for approval routing.
- **Configuration:** checks `{{$json.data.approved}} == yes`
- **Connections:**  
  - True -> **Create SM asset folder**
  - False -> **Adaptive Prompt Builder – IG/YT**
- **Edge cases / failures:**  
  - If Teams response format changes, condition may fail

### Create SM asset folder
- **Type and role:** Creates `SM_Assets` subfolder inside project folder.
- **Connections:** output to **Update sheet: SM carousel data**

### Update sheet: SM carousel data
- **Type and role:** Saves generated caption and five slide texts to the run row.
- **Configuration:** append/update by `Run`
- **Key expressions / variables:** references `$('The IG/YT Specialist').item.json.output.parameters.*`
- **Edge cases / failures:**  
  - Potential mismatch if actual agent output is under `output.*` rather than `output.parameters.*`

### Generate SM slide 1–5
- **Type and role:** `apiTemplateIo`; generates image slides from a template.
- **Configuration:**  
  - Same APITemplate template ID placeholder used in each
  - Replaces template text field `text_1.text` with one slide’s text
- **Connections:**  
  - Each goes to its corresponding download node
- **Edge cases / failures:**  
  - Placeholder template ID not replaced  
  - Template key `text_1.text` must match APITemplate design

### Download carousel image 1–5
- **Type and role:** HTTP Request; downloads generated image binary or metadata endpoint response.
- **Configuration:** URL from `download_url_png`
- **Connections:**  
  - Each goes to matching Drive upload node
- **Edge cases / failures:**  
  - Expired download URL  
  - Response may be binary while later node expects metadata fields too

### Upload SM image 1 to Drive ... Upload SM image 5 to Drive
- **Type and role:** Google Drive uploads of slide images.
- **Configuration:**  
  - Named `sm_carousel_img_1` ... `sm_carousel_img_5`
  - Saved into `SM_Assets` folder
- **Connections:**  
  - Each to a separate input of **Merge all SM carousel images**
- **Edge cases / failures:**  
  - Binary property assumptions  
  - Upload permissions

### Merge all SM carousel images
- **Type and role:** Merge node with 5 inputs.
- **Configuration:** `numberInputs: 5`
- **Connections:** output to **Collect SM image URLs for publishing**
- **Edge cases / failures:**  
  - Merge behavior depends on arriving items; item alignment can be tricky

### Collect SM image URLs for publishing
- **Type and role:** Set node assembling media URLs and caption.
- **Configuration:**  
  - Collects `download_url` from each image download node
  - Stores `post_ext` from sheet caption
  - Runs once
- **Connections:**  
  - Output to **Save SM post text to Drive**
  - Output to **Publish SM carousel via Blotato**
- **Edge cases / failures:**  
  - One field name is mistakenly `=img_url_3` instead of `img_url_3`
  - Uses download node JSON fields, not Drive share URLs
  - If APITemplate URLs expire before Blotato consumes them, publish may fail

### Save SM post text to Drive
- **Type and role:** Stores caption text in Drive.
- **Configuration:** create text file `sm_post_text` in `SM_Assets`
- **Connections:** terminal for archive side

### Publish SM carousel via Blotato
- **Type and role:** Social publishing to another account, likely Instagram.
- **Configuration:**  
  - Uses account ID placeholder
  - Caption includes source URL from **Extract URLs from approval data**
  - Media URLs are comma-separated from set node fields
- **Connections:**  
  - Output to **Wait 30s for SM carousel**
- **Edge cases / failures:**  
  - Account ID placeholder not replaced  
  - Media URL format may not match Blotato expectations  
  - Expired media URLs  
  - Platform is not explicitly specified in this node

### Wait 30s for SM carousel / Check SM carousel post status / Route SM carousel post status
- **Type and role:** publish polling loop.
- **Configuration:**  
  - Wait 30s
  - Get post by `postSubmissionId`
  - Switch on `status`: `published`, `in-progress`, or fallback
- **Connections:**  
  - `published` -> **Notify: SM carousel published**
  - `in-progress` -> loops to **Wait 30s for SM carousel**
  - fallback -> **Notify: SM carousel publish error**
- **Edge cases / failures:**  
  - Indefinite loop if status remains `in-progress`
  - Unhandled statuses like `failed`, `rejected`, `queued`

### Notify: SM carousel published
- **Type and role:** Teams notification.
- **Configuration:** posts success message with public URL
- **Connections:** output to **Update sheet: SM image URLs**

### Notify: SM carousel publish error
- **Type and role:** Teams alert.
- **Configuration:** points user to Drive folder for manual handling

### Update sheet: SM image URLs
- **Type and role:** writes final public social URL back into the run row.
- **Configuration:** append/update by `Run`, column `sm_post_url`

---

## 2.4 LinkedIn Carousel Branch

**Overview:**  
This branch generates LinkedIn carousel copy, sends it for approval, renders it into a PDF carousel using HTML/CSS, stores assets in Drive, and sends a Teams notification once the asset package is ready.

**Nodes Involved:**  
- Adaptive Prompt Builder – LI Carousel  
- The Linkedin Carousel Specialist  
- OpenAI Chat Model4  
- Structured Output Parser2  
- Auto-fixing Output Parser2  
- OpenAI Chat Model5  
- Review LI carousel in Teams  
- LI Carousel Draft Approved?  
- Update sheet: LI carousel data  
- HTML to PDF  
- Upload LI carousel PDF  
- Create LI carousel asset folder  
- Move LI carousel PDF to folder  
- Save LI carousel caption to Drive  
- Notify: LI carousel assets ready

### Adaptive Prompt Builder – LI Carousel
- **Type and role:** Code node for CREATE/REPAIR inputs.
- **Configuration:**  
  - Reads source topics from sheet
  - Tries to read feedback from node `If1)` which does not exist
  - Pulls previous output from **The Linkedin Carousel Specialist**
- **Connections:**  
  - Input from **Update sheet: content pillars**
  - Loopback from **LI Carousel Draft Approved?** false path
  - Output to **The Linkedin Carousel Specialist**
- **Edge cases / failures:**  
  - Broken feedback node reference likely prevents clean revision loop

### The Linkedin Carousel Specialist
- **Type and role:** Agent producing LinkedIn caption, five 2-word hooks, and five slide texts.
- **Configuration:**  
  - CREATE vs REPAIR logic inside prompt
  - Hooks must be exactly 2 words
  - Slide texts max 40 words
- **Connections:**  
  - Output to **Review LI carousel in Teams**
- **Edge cases / failures:**  
  - Hook length constraint may be violated and need parser repair  
  - Revision loop depends on adaptive builder correctness

### OpenAI Chat Model4 / Structured Output Parser2 / Auto-fixing Output Parser2 / OpenAI Chat Model5
- **Type and role:** LLM + structured schema + repair stack for this branch.
- **Schema fields:** `summary`, `linkedin_post_text`, `hook1`–`hook5`, `topic1`–`topic5`

### Review LI carousel in Teams
- **Type and role:** human review form in Teams.
- **Configuration:**  
  - Displays caption and slide content with hooks
  - Approval radio + feedback field
- **Connections:** to **LI Carousel Draft Approved?**

### LI Carousel Draft Approved?
- **Type and role:** approval router.
- **Configuration:** true if `data.approved` equals `yes`
- **Connections:**  
  - True -> **Update sheet: LI carousel data**
  - False -> **Adaptive Prompt Builder – LI Carousel**

### Update sheet: LI carousel data
- **Type and role:** stores combined hook + topic fields and caption.
- **Configuration:**  
  - Writes `li_carousel_text_1...5`
  - Writes `linkedin_post_text`
  - Combines hooks and topics with underscores
- **Edge cases / failures:**  
  - Formatting inconsistency: first value uses `" _ "` while others use `"_"`

### HTML to PDF
- **Type and role:** community node `n8n-nodes-htmlcsstopdf.htmlcsstopdf`; renders the carousel PDF.
- **Configuration:**  
  - Custom CSS for 1080x1350 slide pages
  - HTML references `The Linkedin Carousel Specialist` output for hooks and topic text
  - Uses placeholder logo/profile URLs and placeholder author identity
  - Output format is `file`
- **Connections:** output to **Upload LI carousel PDF**
- **Version-specific requirements:** self-hosted n8n typically needed for this community node
- **Edge cases / failures:**  
  - External image URLs may fail to load in rendering environment  
  - Placeholder branding not replaced  
  - Fonts from Google may not load in restricted environments

### Upload LI carousel PDF
- **Type and role:** uploads generated PDF to Drive root.
- **Connections:** to **Create LI carousel asset folder**

### Create LI carousel asset folder
- **Type and role:** creates `LI_carousel_Assets` inside project folder.
- **Connections:** to **Move LI carousel PDF to folder**

### Move LI carousel PDF to folder
- **Type and role:** moves uploaded PDF into branch asset folder.
- **Connections:** to **Save LI carousel caption to Drive**

### Save LI carousel caption to Drive
- **Type and role:** saves caption as text file in the asset folder.
- **Connections:** to **Notify: LI carousel assets ready**

### Notify: LI carousel assets ready
- **Type and role:** Teams notification that assets are ready in Drive.
- **Configuration:** includes direct link to the asset folder
- **Edge cases / failures:**  
  - This branch does not publish via Blotato despite the sticky note summary suggesting publishing/status tracking

---

## 2.5 LinkedIn Text Post Branch

**Overview:**  
This branch creates a standalone LinkedIn text post, routes it through a Teams approval loop, stores the approved text in Drive, publishes it via Blotato, and tracks the resulting public URL.

**Nodes Involved:**  
- Adaptive Prompt Builder – LI Text Post  
- The Linkedin Post Specialist  
- OpenAI Chat Model6  
- Structured Output Parser3  
- Auto-fixing Output Parser3  
- OpenAI Chat Model7  
- Review LI text post in Teams  
- LI Text Post Draft Approved?  
- Update sheet: LI text post data  
- Create LI text post asset folder  
- Save LI text post to Drive  
- Publish LI text post via Blotato  
- Wait 40s for LI text post  
- Check LI text post status  
- Route LI text post status  
- Notify: LI text post published  
- Notify: LI text post error  
- Update sheet: LI carousel hooks

### Adaptive Prompt Builder – LI Text Post
- **Type and role:** Code node for create/repair payloads.
- **Configuration:**  
  - Reads source topics
  - Tries fallback feedback from node `If2`, which does not exist
  - Retrieves previous `post` from **The Linkedin Post Specialist**
- **Connections:**  
  - Input from **Update sheet: content pillars**
  - False loop from **LI Text Post Draft Approved?**
  - Output to **The Linkedin Post Specialist**
- **Edge cases / failures:** broken fallback node reference

### The Linkedin Post Specialist
- **Type and role:** Agent generating one LinkedIn post JSON object.
- **Configuration:**  
  - Max 100 words
  - Hook, body with `\n\n`, ending/CTA
  - Has duplicated prompt sections but functionally still aims at one post output
- **Connections:** output to **Review LI text post in Teams**
- **Edge cases / failures:**  
  - Prompt is somewhat redundant and may confuse model  
  - Schema asks for `summary` and `post`; downstream mainly uses `post`

### OpenAI Chat Model6 / Structured Output Parser3 / Auto-fixing Output Parser3 / OpenAI Chat Model7
- **Type and role:** branch model and schema-repair stack.
- **Schema fields:** `summary`, `post`

### Review LI text post in Teams
- **Type and role:** send-and-wait review.
- **Configuration:** displays post text with formatting preserved

### LI Text Post Draft Approved?
- **Type and role:** approval IF node.
- **Connections:**  
  - True -> **Update sheet: LI text post data**
  - False -> **Adaptive Prompt Builder – LI Text Post**

### Update sheet: LI text post data
- **Type and role:** writes `linkedin_post_text` to sheet.

### Create LI text post asset folder / Save LI text post to Drive
- **Type and role:** archive approved post into `LinkedIn_Post_Assets` folder.

### Publish LI text post via Blotato
- **Type and role:** LinkedIn publisher.
- **Configuration:**  
  - Platform `linkedin`
  - Appends source URL from email body
- **Connections:** to **Wait 40s for LI text post**

### Wait 40s for LI text post / Check LI text post status / Route LI text post status
- **Type and role:** publish status polling loop
- **Routing:**  
  - success -> **Notify: LI text post published**
  - in-progress -> wait again
  - fallback -> **Notify: LI text post error**

### Notify: LI text post published
- **Type and role:** Teams success message
- **Issue:** message says “Your Instagram carousel was successfully published,” which is incorrect for this branch
- **Connections:** to **Update sheet: LI carousel hooks**

### Update sheet: LI carousel hooks
- **Type and role:** despite the name, it stores `li_carousel_post_url`.
- **Configuration:**  
  - Uses public URL from **Check LI text post status**
- **Issue:** likely misnamed and likely writes the LinkedIn text post URL into the carousel URL column

### Notify: LI text post error
- **Type and role:** Teams error notification with Drive fallback

---

## 2.6 LinkedIn Media Post Branch

**Overview:**  
This branch creates a LinkedIn media post composed of short image text plus supporting post copy. After approval, it generates an image via APITemplate.io, stores assets in Drive, publishes them through Blotato, and logs the resulting public URL.

**Nodes Involved:**  
- Adaptive Prompt Builder – LI Media Post  
- The Linkedin Post Specialist1  
- OpenAI Chat Model8  
- Structured Output Parser4  
- Auto-fixing Output Parser4  
- OpenAI Chat Model9  
- Review LI media post in Teams  
- LI Media Post Draft Approved?  
- Update sheet: LI media post data  
- Create LI media post asset folder  
- Save LI media post text to Drive  
- Generate LI media post image  
- Download LI media post image  
- Upload LI media post image to Drive  
- Publish LI media post via Blotato  
- Wait 20s for LI media post  
- Check LI media post status  
- Route LI media post status  
- Notify: LI media post published  
- Notify: LI media post error  
- Update sheet: LI media post image data

### Adaptive Prompt Builder – LI Media Post
- **Type and role:** Code node handling CREATE/REPAIR.
- **Configuration:**  
  - Explicitly writes `mode: CREATE` or `REPAIR`
  - Reads source topics
  - Tries feedback fallback from node `If4`, which does not exist
  - Pulls previous `post` and `guiding_statement` from **The Linkedin Post Specialist1**
- **Connections:**  
  - Input from **Update sheet: content pillars**
  - False loop from **LI Media Post Draft Approved?**
  - Output to **The Linkedin Post Specialist1**
- **Edge cases / failures:** broken fallback node reference

### The Linkedin Post Specialist1
- **Type and role:** Agent generating a short guiding statement and post.
- **Configuration:**  
  - CREATE mode uses source topics
  - REPAIR mode uses previous draft + feedback
  - Output schema expects `guiding_statement` and `post`
- **Connections:** output to **Review LI media post in Teams**
- **Edge cases / failures:**  
  - Depends on valid mode value from adaptive builder

### Structured Output Parser4 / Auto-fixing Output Parser4 / OpenAI Chat Model8 / OpenAI Chat Model9
- **Type and role:** branch model + schema-repair pipeline.
- **Issue:** parser schema properties define `guiding_statement` and `post`, but `required` incorrectly lists `summary` and `post`. This mismatch can cause parsing failures or repair instability.

### Review LI media post in Teams
- **Type and role:** human approval form for image text + post copy

### LI Media Post Draft Approved?
- **Type and role:** approval router
- **Connections:**  
  - True -> **Update sheet: LI media post data**
  - False -> **Adaptive Prompt Builder – LI Media Post**

### Update sheet: LI media post data
- **Type and role:** writes post text and image text to Google Sheets.
- **Configuration:**  
  - `linkedin_post_text`
  - `linkedin_media_post_text`
  - `linkedin_media_post_image_text`
- **Connections:** to **Create LI media post asset folder**

### Create LI media post asset folder / Save LI media post text to Drive
- **Type and role:** archives image text + post text in Drive

### Generate LI media post image
- **Type and role:** APITemplate image generation node.
- **Configuration:** writes `linkedin_media_post_image_text` into template field `text_1.text`
- **Connections:** to **Download LI media post image**
- **Edge cases / failures:** template placeholders must exist

### Download LI media post image
- **Type and role:** HTTP request downloading image from APITemplate response.
- **Configuration:** URL from `download_url`
- **Connections:**  
  - To **Publish LI media post via Blotato**
  - To **Upload LI media post image to Drive**
- **Edge cases / failures:** expired URL or binary handling mismatch

### Upload LI media post image to Drive
- **Type and role:** archives generated image in Drive.

### Publish LI media post via Blotato
- **Type and role:** LinkedIn media post publication.
- **Configuration:**  
  - Platform `linkedin`
  - Post text plus source URL
  - Media URL from current item `download_url`
- **Connections:** to **Wait 20s for LI media post**

### Wait 20s for LI media post / Check LI media post status / Route LI media post status
- **Type and role:** polling loop identical to other Blotato branches
- **Routing:**  
  - success -> **Notify: LI media post published**
  - in-progress -> wait again
  - fallback -> **Notify: LI media post error**

### Notify: LI media post published
- **Type and role:** Teams success notice
- **Issue:** message incorrectly says “Your Instagram carousel was successfully published.”

### Update sheet: LI media post image data
- **Type and role:** writes `li_media_post_url` to the run row.

### Notify: LI media post error
- **Type and role:** Teams failure alert with Drive fallback

---

## 2.7 Documentation and Sticky Notes

**Overview:**  
These nodes are not executed as business logic but document the workflow visually inside n8n. They explain the architecture, setup, and branch purposes.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6

### Sticky Notes
- **Type and role:** `n8n-nodes-base.stickyNote`; visual inline documentation
- **Configuration:** contains setup, architectural summary, and branch descriptions
- **Connections:** none
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for repurposing emails |  | Download Email Attachments | ## How it works<br>1. **Gmail Trigger** watches for emails with a "Content Repurposing" subject<br>2. The attachment is extracted, saved to Google Drive in a date-stamped subfolder<br>3. An **AI Strategist** agent extracts 5 content pillars from the full text<br>4. Content pillars are logged in Google Sheets<br>5. Three parallel AI specialist branches create: (a) IG/YT carousel + caption, (b) LinkedIn carousel, (c) LinkedIn text post, (d) LinkedIn media post<br>6. Each branch sends a draft to Microsoft Teams for human review (approve / reject with a feedback loop)<br>7. On approval: images are generated via APITemplate.io, assets saved to Google Drive, and posts published via Blotato<br>## Setup steps<br>1. Import the workflow and install the community nodes: `@blotato/n8n-nodes-blotato`, `n8n-nodes-htmlcsstopdf` (self-hosted n8n only)<br>2. Configure all required credentials: Gmail OAuth2, Google Drive OAuth2, Google Sheets OAuth2, OpenAI API, Microsoft Teams OAuth2, APITemplate.io API, Blotato API, HTML-to-PDF API<br>3. Set your Google Drive folder IDs, Google Sheets document ID, and APITemplate.io template IDs in the relevant nodes<br>4. Update the Microsoft Teams channel / chat IDs for approval notifications<br>5. Customize the AI agent prompts if needed, then activate the workflow<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Download Email Attachments | n8n-nodes-base.gmail | Fetch full email and download attachment | Gmail Trigger | Extract from File | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Extract from File | n8n-nodes-base.extractFromFile | Extract text from attachment | Download Email Attachments | Save original content to Drive | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Save original content to Drive | n8n-nodes-base.googleDrive | Save extracted text as file in Drive | Extract from File | Create project subfolder | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Create project subfolder | n8n-nodes-base.googleDrive | Create run folder in Drive | Save original content to Drive | Move original to project folder | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Move original to project folder | n8n-nodes-base.googleDrive | Move source file into project folder | Create project subfolder | Extract URLs from approval data | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Extract URLs from approval data | n8n-nodes-base.code | Extract source URL from email body | Move original to project folder | Log run metadata in Google Sheets | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| Log run metadata in Google Sheets | n8n-nodes-base.googleSheets | Append initial run tracking row | Extract URLs from approval data | The Repurpose Strategist | ## How it works ...<br>## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main model for content pillar extraction |  | The Repurpose Strategist | ## Story Teller Agent – The Repurpose Strategist<br>> The core AI agent. Takes the full extracted text and distills it into 5 Content Pillars — each with a hook headline, deep-dive summary, viral factor, and key quote. Uses a Structured Output Parser with Auto-fixing to guarantee valid JSON. An If-node validates that all 5 themes are populated; if not, the agent retries automatically. The output feeds all four downstream content branches in parallel. |
| OpenAI Chat Model (parser) | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for strategist parser |  | Auto-fixing Output Parser | ## Story Teller Agent – The Repurpose Strategist ... |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for strategist output |  | Auto-fixing Output Parser | ## Story Teller Agent – The Repurpose Strategist ... |
| Auto-fixing Output Parser | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair malformed strategist JSON | Structured Output Parser, OpenAI Chat Model (parser) | The Repurpose Strategist | ## Story Teller Agent – The Repurpose Strategist ... |
| The Repurpose Strategist | @n8n/n8n-nodes-langchain.agent | Extract 5 content pillars from source text | Log run metadata in Google Sheets | Validate content pillars exist | ## Story Teller Agent – The Repurpose Strategist ... |
| Validate content pillars exist | n8n-nodes-base.if | Ensure all 5 topics exist and are non-empty | The Repurpose Strategist | Update sheet: content pillars; The Repurpose Strategist | ## Story Teller Agent – The Repurpose Strategist ... |
| Update sheet: content pillars | n8n-nodes-base.googleSheets | Store pillar outputs in tracking sheet | Validate content pillars exist | Adaptive Prompt Builder – LI Media Post; Adaptive Prompt Builder – LI Text Post; Adaptive Prompt Builder – LI Carousel; Adaptive Prompt Builder – IG/YT | ## Story Teller Agent – The Repurpose Strategist ... |
| Adaptive Prompt Builder – IG/YT | n8n-nodes-base.code | Build CREATE/REPAIR payload for social carousel branch | Update sheet: content pillars; IG/YT Draft Approved? | The IG/YT Specialist | ## Instagram Assets : Carousels<br>> The IG/YT Specialist agent writes a caption and 5 carousel slide texts. An Adaptive Prompt Builder (JS node) switches between CREATE mode (first run, using Content Pillars as source) and REPAIR mode (revision loop, applying user feedback to the previous draft). After Teams approval, APITemplate.io generates 5 carousel images, which are downloaded, uploaded to Drive, merged, and published to Instagram via Blotato. A Switch node tracks publishing status and sends success/error notifications back to Teams. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory buffer for IG/YT specialist |  | The IG/YT Specialist | ## Instagram Assets : Carousels ... |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main model for IG/YT specialist |  | The IG/YT Specialist | ## Instagram Assets : Carousels ... |
| OpenAI Chat Model3 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for IG/YT parser |  | Auto-fixing Output Parser1 | ## Instagram Assets : Carousels ... |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce social carousel schema |  | Auto-fixing Output Parser1 | ## Instagram Assets : Carousels ... |
| Auto-fixing Output Parser1 | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair social carousel JSON | Structured Output Parser1, OpenAI Chat Model3 | The IG/YT Specialist | ## Instagram Assets : Carousels ... |
| The IG/YT Specialist | @n8n/n8n-nodes-langchain.agent | Generate social caption and 5 one-liner slides | Adaptive Prompt Builder – IG/YT | Review SM carousel in Teams | ## Instagram Assets : Carousels ... |
| Review SM carousel in Teams | n8n-nodes-base.microsoftTeams | Human review form for social carousel | The IG/YT Specialist | IG/YT Draft Approved? | ## Instagram Assets : Carousels ... |
| IG/YT Draft Approved? | n8n-nodes-base.if | Approve/reject social carousel | Review SM carousel in Teams | Create SM asset folder; Adaptive Prompt Builder – IG/YT | ## Instagram Assets : Carousels ... |
| Create SM asset folder | n8n-nodes-base.googleDrive | Create Drive folder for social assets | IG/YT Draft Approved? | Update sheet: SM carousel data | ## Instagram Assets : Carousels ... |
| Update sheet: SM carousel data | n8n-nodes-base.googleSheets | Save approved social caption and slides | Create SM asset folder | Generate SM slide 1; Generate SM slide 3; Generate SM slide 4; Generate SM slide 5; Generate SM slide 2 | ## Instagram Assets : Carousels ... |
| Generate SM slide 1 | n8n-nodes-base.apiTemplateIo | Render slide 1 image | Update sheet: SM carousel data | Download carousel image 1 | ## Instagram Assets : Carousels ... |
| Generate SM slide 2 | n8n-nodes-base.apiTemplateIo | Render slide 2 image | Update sheet: SM carousel data | Download carousel image 2 | ## Instagram Assets : Carousels ... |
| Generate SM slide 3 | n8n-nodes-base.apiTemplateIo | Render slide 3 image | Update sheet: SM carousel data | Download carousel image 3 | ## Instagram Assets : Carousels ... |
| Generate SM slide 4 | n8n-nodes-base.apiTemplateIo | Render slide 4 image | Update sheet: SM carousel data | Download carousel image 4 | ## Instagram Assets : Carousels ... |
| Generate SM slide 5 | n8n-nodes-base.apiTemplateIo | Render slide 5 image | Update sheet: SM carousel data | Download carousel image 5 | ## Instagram Assets : Carousels ... |
| Download carousel image 1 | n8n-nodes-base.httpRequest | Download generated social slide 1 | Generate SM slide 1 | Upload SM image 1 to Drive | ## Instagram Assets : Carousels ... |
| Download carousel image 2 | n8n-nodes-base.httpRequest | Download generated social slide 2 | Generate SM slide 2 | Upload SM image 2 to Drive | ## Instagram Assets : Carousels ... |
| Download carousel image 3 | n8n-nodes-base.httpRequest | Download generated social slide 3 | Generate SM slide 3 | Upload SM image 3 to Drive | ## Instagram Assets : Carousels ... |
| Download carousel image 4 | n8n-nodes-base.httpRequest | Download generated social slide 4 | Generate SM slide 4 | Upload SM image 4 to Drive | ## Instagram Assets : Carousels ... |
| Download carousel image 5 | n8n-nodes-base.httpRequest | Download generated social slide 5 | Generate SM slide 5 | Upload SM image 5 to Drive | ## Instagram Assets : Carousels ... |
| Upload SM image 1 to Drive | n8n-nodes-base.googleDrive | Archive social slide 1 in Drive | Download carousel image 1 | Merge all SM carousel images | ## Instagram Assets : Carousels ... |
| Upload SM image 2 to Drive | n8n-nodes-base.googleDrive | Archive social slide 2 in Drive | Download carousel image 2 | Merge all SM carousel images | ## Instagram Assets : Carousels ... |
| Upload SM image 3 to Drive | n8n-nodes-base.googleDrive | Archive social slide 3 in Drive | Download carousel image 3 | Merge all SM carousel images | ## Instagram Assets : Carousels ... |
| Upload SM image 4 to Drive | n8n-nodes-base.googleDrive | Archive social slide 4 in Drive | Download carousel image 4 | Merge all SM carousel images | ## Instagram Assets : Carousels ... |
| Upload SM image 5 to Drive | n8n-nodes-base.googleDrive | Archive social slide 5 in Drive | Download carousel image 5 | Merge all SM carousel images | ## Instagram Assets : Carousels ... |
| Merge all SM carousel images | n8n-nodes-base.merge | Join all 5 social image streams | Upload SM image 1 to Drive; Upload SM image 2 to Drive; Upload SM image 3 to Drive; Upload SM image 4 to Drive; Upload SM image 5 to Drive | Collect SM image URLs for publishing | ## Instagram Assets : Carousels ... |
| Collect SM image URLs for publishing | n8n-nodes-base.set | Assemble image URLs and caption for publish | Merge all SM carousel images | Save SM post text to Drive; Publish SM carousel via Blotato | ## Instagram Assets : Carousels ... |
| Save SM post text to Drive | n8n-nodes-base.googleDrive | Archive social caption text | Collect SM image URLs for publishing |  | ## Instagram Assets : Carousels ... |
| Publish SM carousel via Blotato | @blotato/n8n-nodes-blotato.blotato | Publish social carousel | Collect SM image URLs for publishing | Wait 30s for SM carousel | ## Instagram Assets : Carousels ... |
| Wait 30s for SM carousel | n8n-nodes-base.wait | Delay before social publish status check | Publish SM carousel via Blotato; Route SM carousel post status | Check SM carousel post status | ## Instagram Assets : Carousels ... |
| Check SM carousel post status | @blotato/n8n-nodes-blotato.blotato | Poll social post submission status | Wait 30s for SM carousel | Route SM carousel post status | ## Instagram Assets : Carousels ... |
| Route SM carousel post status | n8n-nodes-base.switch | Route social publish success/in-progress/error | Check SM carousel post status | Notify: SM carousel published; Wait 30s for SM carousel; Notify: SM carousel publish error | ## Instagram Assets : Carousels ... |
| Notify: SM carousel published | n8n-nodes-base.microsoftTeams | Send Teams success message for social post | Route SM carousel post status | Update sheet: SM image URLs | ## Instagram Assets : Carousels ... |
| Notify: SM carousel publish error | n8n-nodes-base.microsoftTeams | Send Teams error message for social post | Route SM carousel post status |  | ## Instagram Assets : Carousels ... |
| Update sheet: SM image URLs | n8n-nodes-base.googleSheets | Log final social public URL | Notify: SM carousel published |  | ## Instagram Assets : Carousels ... |
| Adaptive Prompt Builder – LI Carousel | n8n-nodes-base.code | Build CREATE/REPAIR payload for LinkedIn carousel | Update sheet: content pillars; LI Carousel Draft Approved? | The Linkedin Carousel Specialist | ## Linkedin Assets : Carousels<br>> The LinkedIn Carousel Specialist agent creates a companion post text plus 5 hook-headlines (2 words each) and 5 slide body texts (max 40 words). Same Adaptive Prompt Builder pattern for the feedback loop. After approval, the content is rendered into a multi-page HTML carousel, converted to PDF via HTML-to-PDF, uploaded to Drive, and published to LinkedIn via Blotato with status tracking. |
| OpenAI Chat Model4 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main model for LinkedIn carousel specialist |  | The Linkedin Carousel Specialist | ## Linkedin Assets : Carousels ... |
| OpenAI Chat Model5 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for LinkedIn carousel parser |  | Auto-fixing Output Parser2 | ## Linkedin Assets : Carousels ... |
| Structured Output Parser2 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce LinkedIn carousel schema |  | Auto-fixing Output Parser2 | ## Linkedin Assets : Carousels ... |
| Auto-fixing Output Parser2 | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair LinkedIn carousel JSON | Structured Output Parser2, OpenAI Chat Model5 | The Linkedin Carousel Specialist | ## Linkedin Assets : Carousels ... |
| The Linkedin Carousel Specialist | @n8n/n8n-nodes-langchain.agent | Generate hooks, slides, and caption for LinkedIn carousel | Adaptive Prompt Builder – LI Carousel | Review LI carousel in Teams | ## Linkedin Assets : Carousels ... |
| Review LI carousel in Teams | n8n-nodes-base.microsoftTeams | Human review form for LinkedIn carousel | The Linkedin Carousel Specialist | LI Carousel Draft Approved? | ## Linkedin Assets : Carousels ... |
| LI Carousel Draft Approved? | n8n-nodes-base.if | Approve/reject LinkedIn carousel draft | Review LI carousel in Teams | Update sheet: LI carousel data; Adaptive Prompt Builder – LI Carousel | ## Linkedin Assets : Carousels ... |
| Update sheet: LI carousel data | n8n-nodes-base.googleSheets | Save approved LinkedIn carousel texts | LI Carousel Draft Approved? | HTML to PDF | ## Linkedin Assets : Carousels ... |
| HTML to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Render multi-page carousel PDF | Update sheet: LI carousel data | Upload LI carousel PDF | ## Linkedin Assets : Carousels ... |
| Upload LI carousel PDF | n8n-nodes-base.googleDrive | Upload generated carousel PDF | HTML to PDF | Create LI carousel asset folder | ## Linkedin Assets : Carousels ... |
| Create LI carousel asset folder | n8n-nodes-base.googleDrive | Create Drive folder for LinkedIn carousel assets | Upload LI carousel PDF | Move LI carousel PDF to folder | ## Linkedin Assets : Carousels ... |
| Move LI carousel PDF to folder | n8n-nodes-base.googleDrive | Move PDF into LinkedIn carousel asset folder | Create LI carousel asset folder | Save LI carousel caption to Drive | ## Linkedin Assets : Carousels ... |
| Save LI carousel caption to Drive | n8n-nodes-base.googleDrive | Save LinkedIn carousel caption text | Move LI carousel PDF to folder | Notify: LI carousel assets ready | ## Linkedin Assets : Carousels ... |
| Notify: LI carousel assets ready | n8n-nodes-base.microsoftTeams | Notify Teams that LinkedIn carousel assets are ready | Save LI carousel caption to Drive |  | ## Linkedin Assets : Carousels ... |
| Adaptive Prompt Builder – LI Text Post | n8n-nodes-base.code | Build CREATE/REPAIR payload for LinkedIn text post | Update sheet: content pillars; LI Text Post Draft Approved? | The Linkedin Post Specialist | ## Linkedin Assets : Text-Post<br>> The LinkedIn Post Specialist agent writes a standalone text post (~100 words, hook-body-CTA structure). The Adaptive Prompt Builder handles CREATE vs. REPAIR mode. After Teams approval, the post text is saved as a Google Doc in Drive, published to LinkedIn via Blotato, and the public URL is logged back into the Google Sheet. Status notifications via Teams. |
| OpenAI Chat Model6 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main model for LinkedIn text post specialist |  | The Linkedin Post Specialist | ## Linkedin Assets : Text-Post ... |
| OpenAI Chat Model7 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for LinkedIn text post parser |  | Auto-fixing Output Parser3 | ## Linkedin Assets : Text-Post ... |
| Structured Output Parser3 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce LinkedIn text post schema |  | Auto-fixing Output Parser3 | ## Linkedin Assets : Text-Post ... |
| Auto-fixing Output Parser3 | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair LinkedIn text post JSON | Structured Output Parser3, OpenAI Chat Model7 | The Linkedin Post Specialist | ## Linkedin Assets : Text-Post ... |
| The Linkedin Post Specialist | @n8n/n8n-nodes-langchain.agent | Generate standalone LinkedIn text post | Adaptive Prompt Builder – LI Text Post | Review LI text post in Teams | ## Linkedin Assets : Text-Post ... |
| Review LI text post in Teams | n8n-nodes-base.microsoftTeams | Human review form for LinkedIn text post | The Linkedin Post Specialist | LI Text Post Draft Approved? | ## Linkedin Assets : Text-Post ... |
| LI Text Post Draft Approved? | n8n-nodes-base.if | Approve/reject LinkedIn text post | Review LI text post in Teams | Update sheet: LI text post data; Adaptive Prompt Builder – LI Text Post | ## Linkedin Assets : Text-Post ... |
| Update sheet: LI text post data | n8n-nodes-base.googleSheets | Save approved LinkedIn text post | LI Text Post Draft Approved? | Create LI text post asset folder | ## Linkedin Assets : Text-Post ... |
| Create LI text post asset folder | n8n-nodes-base.googleDrive | Create Drive folder for LinkedIn text post assets | Update sheet: LI text post data | Save LI text post to Drive | ## Linkedin Assets : Text-Post ... |
| Save LI text post to Drive | n8n-nodes-base.googleDrive | Archive LinkedIn text post in Drive | Create LI text post asset folder | Publish LI text post via Blotato | ## Linkedin Assets : Text-Post ... |
| Publish LI text post via Blotato | @blotato/n8n-nodes-blotato.blotato | Publish LinkedIn text post | Save LI text post to Drive | Wait 40s for LI text post | ## Linkedin Assets : Text-Post ... |
| Wait 40s for LI text post | n8n-nodes-base.wait | Delay before LinkedIn text status check | Publish LI text post via Blotato; Route LI text post status | Check LI text post status | ## Linkedin Assets : Text-Post ... |
| Check LI text post status | @blotato/n8n-nodes-blotato.blotato | Poll LinkedIn text post status | Wait 40s for LI text post | Route LI text post status | ## Linkedin Assets : Text-Post ... |
| Route LI text post status | n8n-nodes-base.switch | Route text-post success/in-progress/error | Check LI text post status | Notify: LI text post published; Wait 40s for LI text post; Notify: LI text post error | ## Linkedin Assets : Text-Post ... |
| Notify: LI text post published | n8n-nodes-base.microsoftTeams | Send Teams success message for LinkedIn text post | Route LI text post status | Update sheet: LI carousel hooks | ## Linkedin Assets : Text-Post ... |
| Notify: LI text post error | n8n-nodes-base.microsoftTeams | Send Teams error message for LinkedIn text post | Route LI text post status |  | ## Linkedin Assets : Text-Post ... |
| Update sheet: LI carousel hooks | n8n-nodes-base.googleSheets | Log final URL from LinkedIn text post branch | Notify: LI text post published |  | ## Linkedin Assets : Text-Post ... |
| Adaptive Prompt Builder – LI Media Post | n8n-nodes-base.code | Build CREATE/REPAIR payload for LinkedIn media post | Update sheet: content pillars; LI Media Post Draft Approved? | The Linkedin Post Specialist1 | ## Linkedin Assets : Media-Post<br>> The LinkedIn Media Post Specialist agent writes a "guiding statement" and a media post text. Same feedback loop pattern via Adaptive Prompt Builder. After approval, an image is generated via APITemplate.io, uploaded to Drive alongside the post text, published to LinkedIn via Blotato, and status is tracked with Teams notifications. The public post URL is logged in the Sheet. |
| OpenAI Chat Model8 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main model for LinkedIn media post specialist |  | The Linkedin Post Specialist1 | ## Linkedin Assets : Media-Post ... |
| OpenAI Chat Model9 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Repair model for LinkedIn media post parser |  | Auto-fixing Output Parser4 | ## Linkedin Assets : Media-Post ... |
| Structured Output Parser4 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce LinkedIn media post schema |  | Auto-fixing Output Parser4 | ## Linkedin Assets : Media-Post ... |
| Auto-fixing Output Parser4 | @n8n/n8n-nodes-langchain.outputParserAutofixing | Repair LinkedIn media post JSON | Structured Output Parser4, OpenAI Chat Model9 | The Linkedin Post Specialist1 | ## Linkedin Assets : Media-Post ... |
| The Linkedin Post Specialist1 | @n8n/n8n-nodes-langchain.agent | Generate LinkedIn media post copy and image statement | Adaptive Prompt Builder – LI Media Post | Review LI media post in Teams | ## Linkedin Assets : Media-Post ... |
| Review LI media post in Teams | n8n-nodes-base.microsoftTeams | Human review form for LinkedIn media post | The Linkedin Post Specialist1 | LI Media Post Draft Approved? | ## Linkedin Assets : Media-Post ... |
| LI Media Post Draft Approved? | n8n-nodes-base.if | Approve/reject LinkedIn media post | Review LI media post in Teams | Update sheet: LI media post data; Adaptive Prompt Builder – LI Media Post | ## Linkedin Assets : Media-Post ... |
| Update sheet: LI media post data | n8n-nodes-base.googleSheets | Save approved LinkedIn media post text and image text | LI Media Post Draft Approved? | Create LI media post asset folder | ## Linkedin Assets : Media-Post ... |
| Create LI media post asset folder | n8n-nodes-base.googleDrive | Create Drive folder for LinkedIn media assets | Update sheet: LI media post data | Save LI media post text to Drive | ## Linkedin Assets : Media-Post ... |
| Save LI media post text to Drive | n8n-nodes-base.googleDrive | Archive media post text and image text | Create LI media post asset folder | Generate LI media post image | ## Linkedin Assets : Media-Post ... |
| Generate LI media post image | n8n-nodes-base.apiTemplateIo | Render image for LinkedIn media post | Save LI media post text to Drive | Download LI media post image | ## Linkedin Assets : Media-Post ... |
| Download LI media post image | n8n-nodes-base.httpRequest | Download rendered media post image | Generate LI media post image | Publish LI media post via Blotato; Upload LI media post image to Drive | ## Linkedin Assets : Media-Post ... |
| Upload LI media post image to Drive | n8n-nodes-base.googleDrive | Archive rendered media post image | Download LI media post image |  | ## Linkedin Assets : Media-Post ... |
| Publish LI media post via Blotato | @blotato/n8n-nodes-blotato.blotato | Publish LinkedIn media post | Download LI media post image | Wait 20s for LI media post | ## Linkedin Assets : Media-Post ... |
| Wait 20s for LI media post | n8n-nodes-base.wait | Delay before LinkedIn media status check | Publish LI media post via Blotato; Route LI media post status | Check LI media post status | ## Linkedin Assets : Media-Post ... |
| Check LI media post status | @blotato/n8n-nodes-blotato.blotato | Poll LinkedIn media post status | Wait 20s for LI media post | Route LI media post status | ## Linkedin Assets : Media-Post ... |
| Route LI media post status | n8n-nodes-base.switch | Route media-post success/in-progress/error | Check LI media post status | Notify: LI media post published; Wait 20s for LI media post; Notify: LI media post error | ## Linkedin Assets : Media-Post ... |
| Notify: LI media post published | n8n-nodes-base.microsoftTeams | Send Teams success message for LinkedIn media post | Route LI media post status | Update sheet: LI media post image data | ## Linkedin Assets : Media-Post ... |
| Notify: LI media post error | n8n-nodes-base.microsoftTeams | Send Teams error message for LinkedIn media post | Route LI media post status |  | ## Linkedin Assets : Media-Post ... |
| Update sheet: LI media post image data | n8n-nodes-base.googleSheets | Log final LinkedIn media post URL | Notify: LI media post published |  | ## Linkedin Assets : Media-Post ... |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## How it works ... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Story Teller Agent – The Repurpose Strategist ... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Instagram Assets : Carousels ... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Linkedin Assets : Carousels ... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Linkedin Assets : Text-Post ... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Linkedin Assets : Media-Post ... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Trigger and documentation<br>> Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the Gmail trigger**
   - Add **Gmail Trigger**
   - Set polling to every hour
   - Disable attachment download here
   - Set Gmail search query to `Content Repurposing`
   - Connect Gmail OAuth2

2. **Download the full email with attachments**
   - Add **Gmail**
   - Operation: `Get`
   - Message ID: `{{$json.id}}`
   - Enable attachment download
   - Connect from **Gmail Trigger**

3. **Extract the attachment text**
   - Add **Extract from File**
   - Operation: `Text`
   - Binary property: `attachment_0`
   - Destination key: `content_text`
   - Connect from **Download Email Attachments**

4. **Save the extracted source text to Google Drive**
   - Add **Google Drive**
   - Operation: `Create From Text`
   - Name: `Original_Content_{{$('Gmail Trigger').item.json.headers.subject}}_{{$('Gmail Trigger').item.json.date}}`
   - Content: `{{$json.content_text}}`
   - Folder: root or temporary location
   - Connect Google Drive OAuth2

5. **Create a project subfolder**
   - Add **Google Drive**
   - Resource: `Folder`
   - Parent folder: your main project folder ID
   - Name: `{{$('Gmail Trigger').item.json.headers.subject}}_{{$('Gmail Trigger').item.json.date}}`
   - Connect from **Save original content to Drive**

6. **Move the source file into the project subfolder**
   - Add **Google Drive**
   - Operation: `Move`
   - File ID from **Save original content to Drive**
   - Folder ID from **Create project subfolder**
   - Connect from **Create project subfolder**

7. **Extract source URLs from the email body**
   - Add **Code**
   - Paste logic that scans `$("Gmail Trigger").all()` for URL matches with a regex
   - Return at least:
     - `extractedUrls`
     - `firstUrl`
   - Connect from **Move original to project folder**

8. **Create metadata tracking in Google Sheets**
   - Add **Google Sheets**
   - Operation: `Append`
   - Spreadsheet: your document ID
   - Sheet: target tab
   - Add columns:
     - `Run`
     - `mail_subject`
     - `mail_date`
     - `original_asset_url`
     - `mail_url`
     - `g_drive_folder_url`
   - Use `Run = {{ Math.floor($now.toSeconds()) }}`
   - Connect from **Extract URLs from approval data**

9. **Build the strategist output parser**
   - Add **Structured Output Parser**
   - Manual schema with required fields:
     - `summary`
     - `topic1` to `topic5`
   - Each topic should be a string

10. **Add parser repair model**
    - Add **OpenAI Chat Model**
    - Model: `gpt-5.1`
    - Use for parser fixing

11. **Add auto-fixing parser**
    - Add **Auto-fixing Output Parser**
    - Connect the structured parser and parser model to it

12. **Add the strategist model**
    - Add another **OpenAI Chat Model**
    - Model: `gpt-5.1`

13. **Create the master strategist agent**
    - Add **AI Agent**
    - Paste the large “Content Strategist / Ghostwriter” prompt
    - Reference source text with `{{$('Extract from File').item.json.content_text}}`
    - Enable output parser
    - Connect:
      - main input from **Log run metadata in Google Sheets**
      - AI model from strategist model
      - output parser from auto-fixing parser
    - Set retries to 5

14. **Validate that all five content pillars exist**
    - Add **If**
    - Check each of `output.topic1` through `output.topic5`:
      - exists
      - not empty
    - Connect from **The Repurpose Strategist**
    - False output loops back to **The Repurpose Strategist**

15. **Store the content pillars in Google Sheets**
    - Add **Google Sheets**
    - Operation: `Append or Update`
    - Match by `Run`
    - Write `topic1`–`topic5`
    - Connect from true output of **Validate content pillars exist**

---

## Build the Instagram / Social Carousel branch

16. **Add the adaptive builder for IG/YT**
   - Add **Code**
   - Build CREATE mode from sheet topics
   - Build REPAIR mode from Teams feedback + previous output
   - Ensure the feedback node reference actually points to your approval IF or Teams node; do not keep the broken placeholder `Feedback Reference`

17. **Add IG/YT output parser**
   - Structured schema:
     - `sm_post_text`
     - `topic1`–`topic5`

18. **Add IG/YT parser fix model and auto-fixer**
   - OpenAI model `gpt-5.1`
   - Auto-fixing parser connected to the structured parser

19. **Add optional memory**
   - Add **Simple Memory**
   - Session key can be run-based or fixed per execution

20. **Create the IG/YT specialist agent**
   - Add **AI Agent**
   - Paste the branch-specific prompt
   - Connect:
     - main input from **Adaptive Prompt Builder – IG/YT**
     - AI model from IG/YT model
     - parser from IG/YT auto-fixer
     - memory from **Simple Memory**

21. **Add Teams review for IG/YT**
   - Add **Microsoft Teams**
   - Operation: `sendAndWait`
   - Resource: `chatMessage`
   - Chat ID: your chat
   - Custom form:
     - radio `approved` yes/no
     - text `Feedback`
   - Message should render caption and 5 slide texts

22. **Add approval IF**
   - Check `{{$json.data.approved}} == yes`
   - True continues
   - False loops back to **Adaptive Prompt Builder – IG/YT**

23. **Create the social asset folder**
   - Add **Google Drive folder**
   - Name: `SM_Assets`
   - Parent: project folder ID

24. **Store approved social data in Google Sheets**
   - Add **Google Sheets append/update**
   - Match on `Run`
   - Write:
     - `sm_post_text`
     - `sm_carousel_text1`–`sm_carousel_text5`
   - Prefer consistent expressions based on actual agent output path

25. **Generate five social slides**
   - Add 5 **APITemplate.io** nodes
   - Use your `imageTemplateId`
   - Override `text_1.text` with each slide text column

26. **Download each generated slide**
   - Add 5 **HTTP Request** nodes
   - URL from each APITemplate `download_url_png`

27. **Upload each slide to Google Drive**
   - Add 5 **Google Drive upload** nodes
   - Save into `SM_Assets`

28. **Merge the five image branches**
   - Add **Merge**
   - Set `numberInputs = 5`

29. **Assemble media URLs**
   - Add **Set**
   - Build:
     - `img_url_1` to `img_url_5`
     - `post_ext`
   - Ensure there is no typo like `=img_url_3`

30. **Save caption text to Drive**
   - Add **Google Drive createFromText**
   - Save `post_ext` into `SM_Assets`

31. **Publish via Blotato**
   - Add **Blotato**
   - Configure account
   - Caption = approved social caption + source URL from extracted email URL
   - Media URLs = comma-separated list of 5 slide URLs

32. **Poll social publication status**
   - Add **Wait** 30 seconds
   - Add **Blotato Get Post**
   - Add **Switch**
   - Route:
     - `published` -> success notice
     - `in-progress` -> back to wait
     - fallback -> error notice

33. **Add Teams notifications**
   - Success: include `publicUrl`
   - Error: include Drive fallback link

34. **Log final social public URL in Google Sheets**
   - Update `sm_post_url` by `Run`

---

## Build the LinkedIn carousel branch

35. **Add adaptive builder for LinkedIn carousel**
   - Add **Code**
   - Build CREATE/REPAIR logic
   - Make sure feedback fallback points to a real node, not `If1)`

36. **Add LinkedIn carousel parser stack**
   - Structured schema:
     - `summary`
     - `linkedin_post_text`
     - `hook1`–`hook5`
     - `topic1`–`topic5`
   - Add repair model and auto-fixer

37. **Create LinkedIn carousel specialist agent**
   - Add **AI Agent**
   - Paste the branch prompt
   - Connect model and parser

38. **Add Teams review**
   - Send caption + hooks + slide texts
   - Include approval radio and feedback field

39. **Add approval IF**
   - True proceeds
   - False loops back to adaptive builder

40. **Store approved LinkedIn carousel data**
   - Update Google Sheets by `Run`
   - Save:
     - `li_carousel_text_1`–`li_carousel_text_5`
     - `linkedin_post_text`

41. **Create HTML carousel renderer**
   - Add **HTML to PDF** community node
   - Paste CSS and HTML
   - Replace:
     - logo URL
     - profile image URL
     - author name
     - handle
   - Inject hooks and topics from the specialist output

42. **Upload and organize PDF**
   - Upload generated file to Drive
   - Create `LI_carousel_Assets` folder under project folder
   - Move PDF into that folder

43. **Save caption text**
   - Save `linkedin_post_text` to the same folder

44. **Notify Teams that assets are ready**
   - Include direct Drive folder link

---

## Build the LinkedIn text-post branch

45. **Add adaptive builder for LinkedIn text post**
   - Add **Code**
   - CREATE from content pillars
   - REPAIR from previous post + feedback
   - Replace broken fallback node `If2` with a real node reference

46. **Add parser stack**
   - Schema:
     - `summary`
     - `post`
   - Add repair model and auto-fixer

47. **Create LinkedIn text post agent**
   - Add **AI Agent**
   - Paste the dedicated prompt
   - Connect model and parser

48. **Add Teams review**
   - Display post body with whitespace preserved
   - Include approval + feedback

49. **Add approval IF**
   - True -> proceed
   - False -> loop back to builder

50. **Store approved post**
   - Update Google Sheets with `linkedin_post_text`

51. **Archive the post**
   - Create `LinkedIn_Post_Assets` folder
   - Save the post as a text file

52. **Publish through Blotato**
   - Platform: `linkedin`
   - Caption = approved post + source URL

53. **Poll publication status**
   - Wait 40 seconds
   - Check Blotato status
   - Switch on `published`, `in-progress`, fallback

54. **Add Teams notifications**
   - Fix the success message so it refers to a LinkedIn text post, not Instagram carousel

55. **Log final LinkedIn text post URL**
   - Update a dedicated sheet column such as `li_text_post_url`
   - Do not reuse the carousel URL column

---

## Build the LinkedIn media-post branch

56. **Add adaptive builder for LinkedIn media post**
   - Add **Code**
   - Emit:
     - `mode`
     - `feedback`
     - source topics
     - previous `post`
     - previous `guiding_statement`
   - Replace broken fallback node `If4` with a real node reference

57. **Add parser stack**
   - Structured schema should correctly require:
     - `guiding_statement`
     - `post`
   - Do not keep the current mismatch that requires `summary`

58. **Create LinkedIn media post specialist**
   - Add **AI Agent**
   - Use mode-based prompt
   - Connect parser and model

59. **Add Teams review**
   - Show guiding statement and post text
   - Add approval + feedback

60. **Add approval IF**
   - True -> proceed
   - False -> loop back

61. **Store approved media post data**
   - Update sheet with:
     - `linkedin_media_post_text`
     - `linkedin_media_post_image_text`

62. **Archive media post assets**
   - Create `LinkedIn_image_Post_Assets` folder
   - Save both image text and post text

63. **Generate the media image**
   - Add **APITemplate.io**
   - Inject `linkedin_media_post_image_text` into template text field

64. **Download the generated image**
   - Add **HTTP Request**
   - Use APITemplate `download_url`

65. **Upload generated image to Drive**
   - Add **Google Drive upload**
   - Save into media post asset folder

66. **Publish media post through Blotato**
   - Platform: `linkedin`
   - Post text = approved text + source URL
   - Media URL = APITemplate image URL

67. **Poll media publication status**
   - Wait 20 seconds
   - Check Blotato submission
   - Switch on status

68. **Add Teams notifications**
   - Fix success message wording so it refers to LinkedIn media post

69. **Log final media post URL**
   - Update `li_media_post_url` in Google Sheets

---

## Credentials and setup checklist

70. **Configure credentials**
   - Gmail OAuth2
   - Google Drive OAuth2
   - Google Sheets OAuth2
   - OpenAI API
   - Microsoft Teams OAuth2
   - APITemplate.io API
   - Blotato API
   - HTML-to-PDF service / community node requirements

71. **Replace all placeholders**
   - `YOUR_GOOGLE_DRIVE_FOLDER_ID`
   - `YOUR_GOOGLE_SHEETS_DOC_ID`
   - `YOUR_TEAMS_CHAT_ID`
   - `YOUR_BLOTATO_ACCOUNT_ID`
   - `YOUR_BLOTATO_ACCOUNT_ID_2`
   - `YOUR_APITEMPLATE_TEMPLATE_ID`
   - branding URLs in HTML

72. **Correct known issues before activation**
   - Fix broken adaptive builder references: `Feedback Reference`, `If1)`, `If2`, `If4`
   - Fix `Structured Output Parser4` required fields
   - Fix `=img_url_3` field name typo
   - Fix LI text-post URL logging column/node naming
   - Fix notification texts mentioning Instagram in LinkedIn branches

73. **Activate and test**
   - Send a test email matching the Gmail query with one attachment
   - Confirm:
     - text extraction works
     - content pillars populate
     - Teams approvals loop correctly
     - generated assets save to Drive
     - final URLs write back to Sheets

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install community nodes: `@blotato/n8n-nodes-blotato`, `n8n-nodes-htmlcsstopdf` | Required for publishing and PDF rendering |
| HTML-to-PDF branch uses external branding assets that are placeholders | Replace `https://example.com/your-logo.png` and `https://example.com/your-profile-photo.png` |
| The visual notes state LinkedIn carousel is published via Blotato, but the actual JSON only creates and stores the PDF assets | Workflow consistency note |
| The workflow description in sticky notes says “Three parallel AI specialist branches” but there are actually four output branches | Architecture clarification |
| Teams review is human-in-the-loop using custom form responses with `approved` and `Feedback` | Important for repair loops |
| Google Sheets acts as the central state store for run metadata, pillars, drafts, and public URLs | Operational design |
| Main setup note from workflow canvas | Trigger the workflow and add new drive folder & g-sheet for the documentation of your social media assets |