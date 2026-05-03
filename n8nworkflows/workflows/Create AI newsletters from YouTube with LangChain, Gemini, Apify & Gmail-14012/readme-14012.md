Create AI newsletters from YouTube with LangChain, Gemini, Apify & Gmail

https://n8nworkflows.xyz/workflows/create-ai-newsletters-from-youtube-with-langchain--gemini--apify---gmail-14012


# Create AI newsletters from YouTube with LangChain, Gemini, Apify & Gmail

# 1. Workflow Overview

This workflow turns a submitted YouTube video into a branded AI-generated newsletter and distributes it to a subscriber list by email. It combines website analysis, transcript scraping, AI summarization, HTML newsletter generation, Google Sheets storage, and Gmail delivery.

Primary use cases:
- Generate branded newsletters from creator/company YouTube videos
- Automatically adapt the newsletter to a brand’s logo and color palette
- Save generated newsletter drafts to Google Sheets
- Send personalized emails to all subscribers from a Google Sheet

## 1.1 Input Reception

The workflow starts from an n8n form where the user provides:
- Brand logo file
- Brand name
- Brand website
- YouTube video link

This form submission fans out into parallel branches for brand analysis and video content extraction.

## 1.2 Brand Style and Asset Extraction

The workflow fetches the brand website twice:
- one branch extracts a logo URL from the website HTML
- another branch asks an AI agent to infer the site’s color theme

These outputs are later merged with the transcript-derived content.

## 1.3 YouTube Content Retrieval

Two Apify-backed HTTP requests are used:
- one retrieves the YouTube transcript
- one retrieves the video thumbnail and canonical/original URL

A code node then combines transcript chunks into one text body.

## 1.4 AI Content Generation

An AI agent rewrites the transcript into a journalistic newsletter-style article. Another AI agent converts the generated content into email-ready HTML using a fixed layout and dynamic text colors based on the brand theme.

## 1.5 Aggregation and Subscriber Distribution

All generated/interpreted assets are merged and aggregated into one payload. The workflow then loads subscriber rows from Google Sheets, loops through them, generates recipient-facing HTML content, stores the result in a draft sheet, and sends it via Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block collects the required user inputs through an n8n form trigger. It is the only entry point of the workflow and launches all downstream branches in parallel.

### Nodes Involved
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`; workflow entry trigger via hosted form/webhook
- **Configuration choices:**
  - Form title: `Newsletter`
  - Required fields:
    - Brand Logo (file, single file)
    - Brand Name
    - Brand Website
    - You Tube Video Link
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `{{$json['Brand Website']}}`
    - `{{$json['You Tube Video Link']}}`
    - `{{$json['Brand Name']}}`
- **Input and output connections:**
  - Entry point: none
  - Outputs to:
    - HTTP Request2
    - You Tube Transcript Scraper
    - You Tube Thumbnail Scraper
    - HTTP Request5
- **Version-specific requirements:**
  - Uses `typeVersion 2.3` of the Form Trigger node
- **Edge cases or potential failure types:**
  - Invalid or unreachable website URL
  - Invalid YouTube URL
  - Uploaded logo file is collected but not actually used later in the workflow
  - Form webhook not active if workflow is inactive or not published correctly
- **Sub-workflow reference:** none

---

## 2.2 Brand Style and Asset Extraction

### Overview
This block analyzes the submitted brand website to derive reusable branding elements: a logo URL and a color palette. It uses both extraction and AI interpretation over fetched HTML.

### Nodes Involved
- HTTP Request5
- Information Extractor
- Google Gemini Chat Model
- HTTP Request2
- AI Agent1
- Structured Output Parser2

### Node Details

#### HTTP Request5
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads website HTML for logo extraction
- **Configuration choices:**
  - URL is taken from the submitted `Brand Website`
  - Default GET request
- **Key expressions or variables used:**
  - `={{ $json['Brand Website'] }}`
- **Input and output connections:**
  - Input from: On form submission
  - Output to: Information Extractor
- **Version-specific requirements:**
  - Uses `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - SSL/TLS issues
  - Timeouts on slow websites
  - Redirect-heavy sites
  - Sites blocking bots or returning anti-bot pages
  - HTML too noisy for effective extraction
- **Sub-workflow reference:** none

#### Information Extractor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`; LLM-assisted structured extraction from website HTML
- **Configuration choices:**
  - Input text: the fetched website HTML data
  - Extracted attribute:
    - `Logo` described as `Logo Image Link`
- **Key expressions or variables used:**
  - `={{ $json.data }}`
- **Input and output connections:**
  - Main input from: HTTP Request5
  - AI language model input from: Google Gemini Chat Model
  - Main output to: Merge
- **Version-specific requirements:**
  - Uses `typeVersion 1.2`
  - Requires a compatible LangChain model connection
- **Edge cases or potential failure types:**
  - Website has no accessible logo URL in HTML
  - Logo appears only via JavaScript rendering and not in static HTML
  - Extractor may return the wrong image if page contains multiple brand assets
- **Sub-workflow reference:** none

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; shared LLM provider for multiple AI nodes
- **Configuration choices:**
  - No advanced options set
  - Uses Google Gemini API credentials
- **Key expressions or variables used:** none directly
- **Input and output connections:**
  - AI model output connected to:
    - AI Agent1
    - AI Agent2
    - Information Extractor
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires valid Google Gemini/PaLM credential setup in n8n
- **Edge cases or potential failure types:**
  - API quota/rate limits
  - Credential misconfiguration
  - Regional/model availability issues
- **Sub-workflow reference:** none

#### HTTP Request2
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads website HTML for AI color analysis
- **Configuration choices:**
  - URL taken from `Brand Website`
  - Default GET request
- **Key expressions or variables used:**
  - `={{ $json['Brand Website'] }}`
- **Input and output connections:**
  - Input from: On form submission
  - Output to: AI Agent1
- **Version-specific requirements:**
  - Uses `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - Same as HTTP Request5
  - Different output content from the same site due to redirects or bot behavior
- **Sub-workflow reference:** none

#### AI Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; infers brand color palette from site HTML
- **Configuration choices:**
  - Prompt input is the website HTML data
  - System instruction asks the model to output:
    - `primary_color`
    - `secondary_color`
    - `accent_color`
    - `background_color`
  - Structured output enabled
- **Key expressions or variables used:**
  - `={{ $json.data }}`
- **Input and output connections:**
  - Main input from: HTTP Request2
  - AI language model input from: Google Gemini Chat Model
  - AI output parser input from: Structured Output Parser2
  - Main output to: Merge
- **Version-specific requirements:**
  - Uses `typeVersion 2.2`
  - Requires a connected LLM and output parser
- **Edge cases or potential failure types:**
  - Model returns invalid hex/color names
  - Website HTML does not expose enough visual clues
  - Output parser validation may fail if the AI response is not schema compliant
- **Sub-workflow reference:** none

#### Structured Output Parser2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; constrains AI Agent1 output to a color-theme JSON object
- **Configuration choices:**
  - Expected schema example contains:
    - `primary_color`
    - `secondary_color`
    - `accent_color`
    - `background_color`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output parser connection to: AI Agent1
- **Version-specific requirements:**
  - Uses `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - AI response includes prose instead of strict object data
  - Missing required keys or malformed values
- **Sub-workflow reference:** none

---

## 2.3 YouTube Content Retrieval

### Overview
This block retrieves the video transcript and thumbnail metadata from Apify APIs. The transcript chunks are merged into one continuous text body, and thumbnail metadata is normalized for later HTML rendering.

### Nodes Involved
- You Tube Transcript Scraper
- Code in JavaScript
- You Tube Thumbnail Scraper
- Edit Fields

### Node Details

#### You Tube Transcript Scraper
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls an Apify actor to retrieve transcript data
- **Configuration choices:**
  - POST request to:
    - `https://api.apify.com/v2/acts/faVsWy9VTSNVIhWpR/run-sync-get-dataset-items`
  - JSON body contains:
    - `videoUrl` from the submitted YouTube link
  - Headers:
    - `Accept: application/json`
    - `Authorization: Bearer <API Key>`
- **Key expressions or variables used:**
  - `{{ $json['You Tube Video Link'] }}`
- **Input and output connections:**
  - Input from: On form submission
  - Output to: Code in JavaScript
- **Version-specific requirements:**
  - Uses `typeVersion 4.2`
  - Requires replacing `<API Key>` with a valid Apify token
- **Edge cases or potential failure types:**
  - Missing/invalid API token
  - Apify actor unavailable or changed
  - Transcript unavailable for the video
  - YouTube restrictions, private videos, region restrictions
  - Rate limits or long-running actor timeouts
- **Sub-workflow reference:** none

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`; consolidates transcript chunks into one text field
- **Configuration choices:**
  - Reads `data` from the input item
  - Maps over transcript chunks and concatenates `text` values
  - Outputs a single field: `combinedText`
- **Key expressions or variables used:**
  - Uses `$input.item.json.data`
- **Input and output connections:**
  - Input from: You Tube Transcript Scraper
  - Output to: AI Agent2
- **Version-specific requirements:**
  - Uses `typeVersion 2`
- **Edge cases or potential failure types:**
  - Input shape differs from expected Apify result
  - `data` missing or not an array
  - Transcript chunks missing `text`
- **Sub-workflow reference:** none

#### You Tube Thumbnail Scraper
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls an Apify actor to retrieve thumbnail/original video metadata
- **Configuration choices:**
  - POST request to:
    - `https://api.apify.com/v2/acts/gLdh4fh3Q9Rqcmqdf/run-sync-get-dataset-items`
  - JSON body contains `video_urls` array with the submitted URL
  - Headers:
    - `Accept: application/json`
    - `Authorization: Bearer <API Key>`
- **Key expressions or variables used:**
  - `{{ $json['You Tube Video Link'] }}`
- **Input and output connections:**
  - Input from: On form submission
  - Output to: Edit Fields
- **Version-specific requirements:**
  - Uses `typeVersion 4.2`
  - Requires replacing `<API Key>` with a valid Apify token
- **Edge cases or potential failure types:**
  - Same token/availability issues as transcript scraper
  - Returned object may not contain `thumbnail` or `original_url`
- **Sub-workflow reference:** none

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`; normalizes thumbnail scraper output into the exact field names used later
- **Configuration choices:**
  - Sets:
    - `Thumbnail = {{$json.thumbnail}}`
    - `You Tube Video Link = {{$json.original_url}}`
- **Key expressions or variables used:**
  - `={{ $json.thumbnail }}`
  - `={{ $json.original_url }}`
- **Input and output connections:**
  - Input from: You Tube Thumbnail Scraper
  - Output to: Merge
- **Version-specific requirements:**
  - Uses `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Missing `thumbnail`
  - Missing `original_url`
  - Field naming with spaces can be awkward for maintenance
- **Sub-workflow reference:** none

---

## 2.4 AI Content Generation

### Overview
This block transforms the raw transcript into newsletter content, then into styled HTML email output with a structured subject/body response. It is the core AI generation section of the workflow.

### Nodes Involved
- AI Agent2
- Merge
- Aggregate
- Loop Over Items
- Convert Newsletter to HTML (AI)
- Structured Output Parser1
- Google Gemini Chat Model1

### Node Details

#### AI Agent2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; converts the raw transcript into a polished journalistic article/newsletter section
- **Configuration choices:**
  - Input text: `Transcript: {{ $json.combinedText }}`
  - System prompt defines:
    - journalist tone
    - transcript cleaning
    - narrative extraction
    - no hallucination
    - no emojis
- **Key expressions or variables used:**
  - `=Transcript:  {{ $json.combinedText }}`
- **Input and output connections:**
  - Main input from: Code in JavaScript
  - AI language model input from: Google Gemini Chat Model
  - Main output to: Merge
- **Version-specific requirements:**
  - Uses `typeVersion 3`
- **Edge cases or potential failure types:**
  - Transcript too long for model context window
  - Hallucination risk despite instructions
  - Output not clearly structured into “three newsletter sections” even though downstream prompt assumes multi-section content
- **Sub-workflow reference:** none

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`; combines four parallel result streams
- **Configuration choices:**
  - `numberInputs: 4`
  - Receives:
    1. logo extraction
    2. color theme
    3. transcript-derived article
    4. thumbnail/video metadata
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Inputs from:
    - Information Extractor
    - AI Agent1
    - AI Agent2
    - Edit Fields
  - Output to: Aggregate
- **Version-specific requirements:**
  - Uses `typeVersion 3.2`
- **Edge cases or potential failure types:**
  - One branch returns no items, causing merge behavior to fail or stall
  - Mismatched item counts across branches
- **Sub-workflow reference:** none

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`; flattens all merged items into a single array field for easier downstream access
- **Configuration choices:**
  - Aggregates all item data
  - Destination field: `output`
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `$('Aggregate').item.json.output[...]`
- **Input and output connections:**
  - Input from: Merge
  - Output to: Get row(s) in sheet
- **Version-specific requirements:**
  - Uses `typeVersion 1`
- **Edge cases or potential failure types:**
  - Assumes a stable item ordering in the aggregated array:
    - `[0]` logo
    - `[1]` color palette
    - `[2]` article content
    - `[3]` thumbnail/video metadata
  - If merge ordering changes, downstream HTML generation breaks silently
- **Sub-workflow reference:** none

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through subscriber records
- **Configuration choices:**
  - No explicit batch size configured
  - `executeOnce: true`
- **Key expressions or variables used:**
  - Downstream email node references:
    - `$('Loop Over Items').item.json.Email`
- **Input and output connections:**
  - Input from: Get row(s) in sheet
  - Output 1 loops back from Gmail completion
  - Output 2 goes to: Convert Newsletter to HTML (AI)
- **Version-specific requirements:**
  - Uses `typeVersion 3`
- **Edge cases or potential failure types:**
  - Misunderstanding of Split In Batches outputs can cause skipped recipients
  - `executeOnce: true` can be surprising depending on expected per-item behavior
  - If subscriber sheet is empty, nothing is sent
- **Sub-workflow reference:** none

#### Convert Newsletter to HTML (AI)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; generates structured output containing email subject and HTML body
- **Configuration choices:**
  - Input text:
    - `Data: {{ $('Aggregate').item.json.output[2].output }}`
  - System prompt enforces:
    - fixed HTML email template
    - dynamic text colors only
    - logo insertion
    - video thumbnail insertion with clickable link
    - recipient name greeting
    - sources section
    - conclusion
    - max 1000 words
  - Uses structured output parser
- **Key expressions or variables used:**
  - `$('Aggregate').item.json.output[0].output.Logo`
  - `$('Aggregate').item.json.output[1].output.primary_color`
  - `$('Aggregate').item.json.output[1].output.secondary_color`
  - `$('Aggregate').item.json.output[1].output.accent_color`
  - `$('Aggregate').item.json.output[3]['You Tube Video Link']`
  - `$('Aggregate').item.json.output[3].Thumbnail`
  - `$('On form submission').item.json['Brand Name']`
  - `$json.Name`
  - `$now.format('yyyy')`
  - `$now.format('yyyy-MM-dd')`
- **Input and output connections:**
  - Main input from: Loop Over Items
  - AI language model input from: Google Gemini Chat Model1
  - AI output parser input from: Structured Output Parser1
  - Main outputs to:
    - Sending Emails to all the Subscribers
    - Save Newsletter Draft in Google Sheet1
- **Version-specific requirements:**
  - Uses `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - If subscriber rows lack a `Name` field, greeting renders blank
  - If aggregated array indices shift, template values point to wrong data
  - AI may produce invalid HTML or omit required sections
  - Sources requirement may be impossible if AI Agent2 output lacks URLs
  - Subject/body parse fails if output parser schema not matched exactly
- **Sub-workflow reference:** none

#### Structured Output Parser1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces `subject` and `content` output keys
- **Configuration choices:**
  - Manual schema:
    - `subject` string
    - `content` string
  - Both required
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output parser connection to: Convert Newsletter to HTML (AI)
- **Version-specific requirements:**
  - Uses `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - AI may return markdown or wrapper text instead of strict object fields
- **Sub-workflow reference:** none

#### Google Gemini Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; dedicated model for final HTML generation
- **Configuration choices:**
  - No custom options configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - AI model output connected to: Convert Newsletter to HTML (AI)
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires valid Gemini API credentials
- **Edge cases or potential failure types:**
  - Same quota/auth/model issues as the other Gemini model node
- **Sub-workflow reference:** none

---

## 2.5 Storage and Email Delivery

### Overview
This block reads subscribers from Google Sheets, stores the generated newsletter draft, and sends the HTML email to each subscriber through Gmail. It closes the batching loop after each send.

### Nodes Involved
- Get row(s) in sheet
- Save Newsletter Draft in Google Sheet1
- Sending Emails to all the Subscribers

### Node Details

#### Get row(s) in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; retrieves subscriber records
- **Configuration choices:**
  - Document: `Newsletter Emails`
  - Sheet: `Subscribers List`
  - Operation appears to be row retrieval using default sheet-read behavior
- **Key expressions or variables used:** none directly
- **Input and output connections:**
  - Input from: Aggregate
  - Output to: Loop Over Items
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Wrong spreadsheet permissions
  - Missing `Email` or `Name` columns in the subscriber sheet
  - Empty sheet
- **Sub-workflow reference:** none

#### Save Newsletter Draft in Google Sheet1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends the generated newsletter output to a draft/archive sheet
- **Configuration choices:**
  - Operation: append
  - Target spreadsheet: `Newsletter Emails`
  - Sheet: `Sheet1`
  - Mapped columns:
    - `Email` = subject
    - `Message` = content
    - `Email Send At` = current timestamp
    - `Email Created At` = current timestamp
- **Key expressions or variables used:**
  - `={{ $json.output.subject }}`
  - `={{ $json.output.content }}`
  - `={{$now}}`
- **Input and output connections:**
  - Input from: Convert Newsletter to HTML (AI)
  - No downstream connection
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Column naming is misleading: subject is stored in a column named `Email`
  - Permission errors
  - HTML content may be large for practical spreadsheet use
- **Sub-workflow reference:** none

#### Sending Emails to all the Subscribers
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the generated HTML email to each subscriber
- **Configuration choices:**
  - Recipient: current loop item `Email`
  - Subject: AI-generated subject
  - Message body: AI-generated HTML content
  - Gmail OAuth2 credential
- **Key expressions or variables used:**
  - `={{ $('Loop Over Items').item.json.Email }}`
  - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.subject }}`
  - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.content }}`
- **Input and output connections:**
  - Input from: Convert Newsletter to HTML (AI)
  - Output loops back to: Loop Over Items
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires Gmail OAuth2 credentials with send permissions
- **Edge cases or potential failure types:**
  - Gmail quota or daily sending limits
  - Invalid subscriber email addresses
  - HTML may be sent as plain content depending on Gmail node behavior/settings
  - OAuth token expiration
- **Sub-workflow reference:** none

---

## 2.6 Documentation and In-Canvas Notes

### Overview
These nodes are documentation-only sticky notes used to explain the workflow inside the n8n canvas. They do not affect execution but provide operational context and setup resources.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation
- **Configuration choices:**
  - Contains full workflow description, operating summary, and external links
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for input block
- **Configuration choices:** describes workflow input form section
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for brand style extraction
- **Configuration choices:** describes website visual identity analysis block
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for transcript retrieval
- **Configuration choices:** describes YouTube transcript and thumbnail fetch block
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; section label for AI processing and delivery
- **Configuration choices:** describes newsletter generation and email sending block
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Collects brand and video inputs via form |  | HTTP Request2; You Tube Transcript Scraper; You Tube Thumbnail Scraper; HTTP Request5 | ## Workflow Input (Form Trigger)<br>This section collects the required inputs from the user through a form submission. |
| HTTP Request2 | n8n-nodes-base.httpRequest | Fetches brand website HTML for color analysis | On form submission | AI Agent1 | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| HTTP Request5 | n8n-nodes-base.httpRequest | Fetches brand website HTML for logo extraction | On form submission | Information Extractor | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Extracts logo URL from website HTML | HTTP Request5 | Merge | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Shared LLM for extraction and transcript analysis |  | AI Agent1; AI Agent2; Information Extractor |  |
| AI Agent1 | @n8n/n8n-nodes-langchain.agent | Infers brand color palette from website HTML | HTTP Request2 | Merge | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Structured Output Parser2 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured color-theme output |  | AI Agent1 | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| You Tube Transcript Scraper | n8n-nodes-base.httpRequest | Retrieves transcript data from Apify | On form submission | Code in JavaScript | ## Fetch YouTube Transcript<br>This section retrieves the transcript and thumbnail from the YouTube video. |
| Code in JavaScript | n8n-nodes-base.code | Combines transcript chunks into one text field | You Tube Transcript Scraper | AI Agent2 | ## Fetch YouTube Transcript<br>This section retrieves the transcript and thumbnail from the YouTube video. |
| AI Agent2 | @n8n/n8n-nodes-langchain.agent | Rewrites transcript into newsletter/article content | Code in JavaScript | Merge | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| You Tube Thumbnail Scraper | n8n-nodes-base.httpRequest | Retrieves YouTube thumbnail and canonical URL from Apify | On form submission | Edit Fields | ## Fetch YouTube Transcript<br>This section retrieves the transcript and thumbnail from the YouTube video. |
| Edit Fields | n8n-nodes-base.set | Renames thumbnail metadata fields | You Tube Thumbnail Scraper | Merge | ## Fetch YouTube Transcript<br>This section retrieves the transcript and thumbnail from the YouTube video. |
| Merge | n8n-nodes-base.merge | Combines logo, colors, article content, and thumbnail data | Information Extractor; AI Agent1; AI Agent2; Edit Fields | Aggregate | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Aggregate | n8n-nodes-base.aggregate | Aggregates merged outputs into one array payload | Merge | Get row(s) in sheet | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Loads subscriber records from Google Sheets | Aggregate | Loop Over Items | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterates through subscriber records | Get row(s) in sheet; Sending Emails to all the Subscribers | Convert Newsletter to HTML (AI) | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Convert Newsletter to HTML (AI) | @n8n/n8n-nodes-langchain.agent | Produces structured subject and HTML newsletter body | Loop Over Items | Sending Emails to all the Subscribers; Save Newsletter Draft in Google Sheet1 | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured subject/content output |  | Convert Newsletter to HTML (AI) | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for final HTML generation |  | Convert Newsletter to HTML (AI) | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Sending Emails to all the Subscribers | n8n-nodes-base.gmail | Sends HTML newsletter to each subscriber via Gmail | Convert Newsletter to HTML (AI) | Loop Over Items | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Save Newsletter Draft in Google Sheet1 | n8n-nodes-base.googleSheets | Appends generated newsletter draft to Google Sheets | Convert Newsletter to HTML (AI) |  | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and setup links |  |  | # Video → Newsletter AI Agent<br><br>This n8n workflow converts a YouTube video into a polished, email-ready newsletter. It scrapes the transcript, extracts a thumbnail/logo and brand color theme, uses multiple AI agents to (1) clean & summarize the transcript into three newsletter sections, (2) convert that content into a styled HTML newsletter (color-aware), then saves the draft to Google Sheets and sends the email to subscribers via Gmail. The flow is optimized for batch sending and brand-consistent HTML output.<br><br>## How it works (step-by-step)<br>1. **Trigger** — `On form submission` accepts Brand Name, Brand Website, and YouTube video link.<br>2. **Site scrape & colour study** — HTTP requests + `Information Extractor` → AI agent derives brand color theme (primary/secondary/accent/background).<br>3. **Transcript retrieval** — Two YouTube transcript scrapers (Apify acts) fetch the video transcript and thumbnail; a small `Code` node merges transcript chunks.<br>4. **Summarization & journalism** — `AI Agent2` (LangChain/Gemini) cleans the transcript, extracts thesis + key points, and writes 3 newsletter sections in a journalistic tone.<br>5. **HTML conversion** — `Convert Newsletter to HTML (AI)` agent applies the fixed layout and injects only text color variables (keeps layout intact) and outputs Subject + HTML body (≤1000 words).<br>6. **Aggregate & merge** — `Merge` + `Aggregate` assemble files, assets, and parsed outputs.<br>7. **Save & send** — Save the email draft to Google Sheets (`Save Newsletter Draft in Google Sheet`) and loop through subscribers from a subscribers sheet; `Sending Emails to all the Subscribers` (Gmail node) sends the HTML to each address in batches.<br>8. **Batching & looping** — `Split In Batches` handles large subscriber lists; `Loop Over Items` triggers the HTML-conversion per recipient batch.<br><br>## Quick Setup Guide<br>👉 [Demo & Setup Video](https://drive.google.com/file/d/1bOfkyDY-Y06pq9ojZzLaGy5Xuvp9LdH7/view?usp=sharing)<br>👉 [Sheet Template](https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing)<br>👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for input section |  |  | ## Workflow Input (Form Trigger)<br>This section collects the required inputs from the user through a form submission. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for branding analysis section |  |  | ## Extract Brand Style & Colors<br>This block analyzes the provided brand website to determine the brand’s visual identity. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for transcript section |  |  | ## Fetch YouTube Transcript<br>This section retrieves the transcript and thumbnail from the YouTube video. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for AI and delivery section |  |  | ## AI Processing, HTML Generation & Email Delivery<br>This section handles the core content generation and delivery of the newsletter. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `Video → Newsletter AI Agent`.

2. **Add a Form Trigger node** named `On form submission`.
   - Set form title to `Newsletter`.
   - Add these fields:
     1. `Brand Logo` as a required single file field
     2. `Brand Name` as a required text field
     3. `Brand Website` as a required text field
     4. `You Tube Video Link` as a required text field

3. **Add an HTTP Request node** named `HTTP Request2`.
   - Method: `GET`
   - URL: `={{ $json['Brand Website'] }}`
   - Connect `On form submission` → `HTTP Request2`

4. **Add a second HTTP Request node** named `HTTP Request5`.
   - Method: `GET`
   - URL: `={{ $json['Brand Website'] }}`
   - Connect `On form submission` → `HTTP Request5`

5. **Add a Google Gemini Chat Model node** named `Google Gemini Chat Model`.
   - Configure Google Gemini API credentials.
   - Keep default options unless you want to specify a model manually.
   - This node will be reused by multiple AI nodes.

6. **Add an Information Extractor node** named `Information Extractor`.
   - Text input: `={{ $json.data }}`
   - Add one attribute:
     - Name: `Logo`
     - Description: `Logo Image Link`
   - Connect `HTTP Request5` → `Information Extractor`
   - Connect `Google Gemini Chat Model` to the Information Extractor’s AI model port

7. **Add a Structured Output Parser node** named `Structured Output Parser2`.
   - Use a JSON example like:
     - `primary_color`
     - `secondary_color`
     - `accent_color`
     - `background_color`

8. **Add an AI Agent node** named `AI Agent1`.
   - Prompt source text: `={{ $json.data }}`
   - Enable defined prompt/system message.
   - System message should instruct the model to analyze website HTML and return:
     - `primary_color`
     - `secondary_color`
     - `accent_color`
     - `background_color`
   - Enable structured output.
   - Connect:
     - `HTTP Request2` → `AI Agent1`
     - `Google Gemini Chat Model` → AI model port of `AI Agent1`
     - `Structured Output Parser2` → output parser port of `AI Agent1`

9. **Add an HTTP Request node** named `You Tube Transcript Scraper`.
   - Method: `POST`
   - URL: `https://api.apify.com/v2/acts/faVsWy9VTSNVIhWpR/run-sync-get-dataset-items`
   - Send body as JSON
   - JSON body:
     - `videoUrl: {{ $json['You Tube Video Link'] }}`
   - Add headers:
     - `Accept: application/json`
     - `Authorization: Bearer YOUR_APIFY_API_KEY`
   - Connect `On form submission` → `You Tube Transcript Scraper`

10. **Add a Code node** named `Code in JavaScript`.
    - Paste logic that:
      - reads `$input.item.json.data`
      - joins each transcript chunk’s `text`
      - returns one item with `combinedText`
    - Connect `You Tube Transcript Scraper` → `Code in JavaScript`

11. **Add an AI Agent node** named `AI Agent2`.
    - Input text: `=Transcript: {{ $json.combinedText }}`
    - Use a system message that instructs the model to:
      - clean transcript filler
      - identify thesis and key points
      - write in a journalistic tone
      - avoid hallucinations
      - avoid emojis
    - Connect:
      - `Code in JavaScript` → `AI Agent2`
      - `Google Gemini Chat Model` → AI model port of `AI Agent2`

12. **Add another HTTP Request node** named `You Tube Thumbnail Scraper`.
    - Method: `POST`
    - URL: `https://api.apify.com/v2/acts/gLdh4fh3Q9Rqcmqdf/run-sync-get-dataset-items`
    - Send body as JSON
    - JSON body should include:
      - `video_urls` array with one object containing the submitted YouTube URL
    - Add headers:
      - `Accept: application/json`
      - `Authorization: Bearer YOUR_APIFY_API_KEY`
    - Connect `On form submission` → `You Tube Thumbnail Scraper`

13. **Add a Set node** named `Edit Fields`.
    - Create fields:
      - `Thumbnail = {{ $json.thumbnail }}`
      - `You Tube Video Link = {{ $json.original_url }}`
    - Connect `You Tube Thumbnail Scraper` → `Edit Fields`

14. **Add a Merge node** named `Merge`.
    - Set number of inputs to `4`
    - Connect these four nodes into it:
      1. `Information Extractor`
      2. `AI Agent1`
      3. `AI Agent2`
      4. `Edit Fields`

15. **Add an Aggregate node** named `Aggregate`.
    - Aggregate mode: aggregate all item data
    - Destination field name: `output`
    - Connect `Merge` → `Aggregate`

16. **Add a Google Sheets node** named `Get row(s) in sheet`.
    - Configure Google Sheets OAuth2 credentials.
    - Select the spreadsheet used for subscribers.
    - Select the `Subscribers List` sheet.
    - Configure it to read rows.
    - The sheet should include at least:
      - `Email`
      - ideally `Name`
    - Connect `Aggregate` → `Get row(s) in sheet`

17. **Add a Split In Batches node** named `Loop Over Items`.
    - Keep default settings if you want the same behavior as the source workflow.
    - Connect `Get row(s) in sheet` → `Loop Over Items`

18. **Add a second Google Gemini Chat Model node** named `Google Gemini Chat Model1`.
    - Configure the same Google Gemini API credential.
    - This isolates final HTML generation from the earlier AI tasks.

19. **Add a Structured Output Parser node** named `Structured Output Parser1`.
    - Create a manual schema:
      - object with required fields:
        - `subject` string
        - `content` string

20. **Add an AI Agent node** named `Convert Newsletter to HTML (AI)`.
    - Connect `Loop Over Items` second/main processing output to this node.
    - Use input text:
      - `Data: {{ $('Aggregate').item.json.output[2].output }}`
    - Add a system message that:
      - keeps a fixed email HTML layout
      - uses only dynamic text colors from the color theme
      - inserts logo and YouTube thumbnail
      - greets recipient by name
      - creates a subject line under 80 characters
      - outputs complete HTML body
    - In the HTML template, reference:
      - logo from `$('Aggregate').item.json.output[0].output.Logo`
      - colors from `$('Aggregate').item.json.output[1].output.*`
      - video URL and thumbnail from `$('Aggregate').item.json.output[3]`
      - brand name from `$('On form submission').item.json['Brand Name']`
      - recipient name from `$json.Name`
    - Connect:
      - `Google Gemini Chat Model1` → AI model port
      - `Structured Output Parser1` → output parser port

21. **Add a Gmail node** named `Sending Emails to all the Subscribers`.
    - Configure Gmail OAuth2 credentials.
    - Set recipient:
      - `={{ $('Loop Over Items').item.json.Email }}`
    - Set subject:
      - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.subject }}`
    - Set message/body:
      - `={{ $('Convert Newsletter to HTML (AI)').item.json.output.content }}`
    - Connect `Convert Newsletter to HTML (AI)` → `Sending Emails to all the Subscribers`

22. **Add a Google Sheets node** named `Save Newsletter Draft in Google Sheet1`.
    - Configure Google Sheets OAuth2 credentials.
    - Operation: `Append`
    - Select the target spreadsheet and sheet for draft storage, here `Sheet1`
    - Map columns as follows:
      - `Email` ← `{{ $json.output.subject }}`
      - `Message` ← `{{ $json.output.content }}`
      - `Email Created At` ← `{{ $now }}`
      - `Email Send At` ← `{{ $now }}`
    - Connect `Convert Newsletter to HTML (AI)` → `Save Newsletter Draft in Google Sheet1`

23. **Close the batch loop** by connecting:
    - `Sending Emails to all the Subscribers` → `Loop Over Items`
   This allows the workflow to continue iterating through subscribers.

24. **Verify sheet structure** before testing:
    - Subscriber sheet should contain:
      - `Email`
      - optional `Name`
    - Draft/archive sheet should contain:
      - `Email`
      - `Message`
      - `Email Created At`
      - `Email Send At`

25. **Replace placeholders and credentials**:
    - Replace both Apify `Authorization: Bearer <API Key>` headers with a real Apify API token
    - Configure:
      - Google Gemini credentials
      - Google Sheets OAuth2 credentials
      - Gmail OAuth2 credentials

26. **Test each branch independently**:
    - Confirm website requests return HTML in `data`
    - Confirm transcript scraper returns transcript chunk array in `data`
    - Confirm thumbnail scraper returns `thumbnail` and `original_url`
    - Confirm AI Agent1 returns four color fields
    - Confirm AI Agent2 returns coherent content
    - Confirm final agent returns valid `subject` and `content`

27. **Test with a small subscriber list first**.
    - Use one or two rows in the subscriber sheet
    - Confirm Gmail rendering is correct
    - Confirm Google Sheets append behavior is correct

28. **Optional hardening recommended for production**:
    - Add validation IF nodes for missing transcript, missing logo, or missing colors
    - Add retry/error handling for HTTP and AI nodes
    - Add HTML sanitization or fallback template
    - Store Apify key in credentials or environment variables instead of hardcoding headers

### Credential Configuration Summary
- **Google Gemini / Google PaLM API**
  - Used by:
    - Google Gemini Chat Model
    - Google Gemini Chat Model1
- **Google Sheets OAuth2**
  - Used by:
    - Get row(s) in sheet
    - Save Newsletter Draft in Google Sheet1
- **Gmail OAuth2**
  - Used by:
    - Sending Emails to all the Subscribers
- **Apify API token**
  - Used in both HTTP Request nodes for transcript and thumbnail scraping

### Data Expectations
- **Form input**
  - Brand Website must be a valid reachable URL
  - YouTube video must have a transcript accessible to the chosen Apify actor
- **Subscriber sheet**
  - Must have `Email`
  - Should have `Name` if personalized greeting is desired
- **Aggregate array order**
  - The final HTML node assumes a strict order in `Aggregate.output`:
    1. Logo extraction
    2. Color theme
    3. AI article content
    4. Thumbnail/video metadata

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1bOfkyDY-Y06pq9ojZzLaGy5Xuvp9LdH7/view?usp=sharing |
| Sheet Template | https://docs.google.com/spreadsheets/d/1hvB5Zif52eCLv_X7E_OifQv9OI5usn-CQ50-TsZTMQA/edit?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Workflow branding note: “Video → Newsletter AI Agent” | Canvas documentation |
| Important implementation note: the uploaded `Brand Logo` form file is currently collected but not used downstream; the workflow instead extracts a logo URL from the website HTML. | Design observation |
| Important implementation note: the draft sheet column named `Email` actually stores the generated subject line, not a recipient email address. | Google Sheets mapping caveat |
| Important implementation note: the final HTML generation depends on positional indexing within `Aggregate.output`, which is fragile if merge ordering changes. | Maintenance caveat |