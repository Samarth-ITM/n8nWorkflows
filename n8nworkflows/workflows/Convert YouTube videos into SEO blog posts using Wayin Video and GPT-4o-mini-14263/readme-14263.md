Convert YouTube videos into SEO blog posts using Wayin Video and GPT-4o-mini

https://n8nworkflows.xyz/workflows/convert-youtube-videos-into-seo-blog-posts-using-wayin-video-and-gpt-4o-mini-14263


# Convert YouTube videos into SEO blog posts using Wayin Video and GPT-4o-mini

# 1. Workflow Overview

This workflow turns a submitted YouTube video URL into a structured SEO blog draft. It uses the Wayin Video API to transcribe the video, waits until the transcription job completes, converts the transcript into a clean text payload with metadata, sends that transcript to GPT-4o-mini to generate a blog post in a strict JSON structure, stores the result in Google Sheets, and finally returns the result through the original webhook response.

Typical use cases include:

- Converting YouTube or other supported video URLs into blog drafts
- Building a content repurposing pipeline for SEO teams
- Automating transcript-to-article generation for content marketing workflows
- Providing machine-readable blog output for later publishing into CMS tools

## 1.1 Input Reception

The workflow starts with a webhook that accepts a POST request containing a video URL in the request body.

## 1.2 Transcription Submission

The submitted URL is sent to Wayin Video’s transcription API, which creates a transcription job and returns a job ID.

## 1.3 Polling for Completion

The workflow repeatedly checks the transcription job status every 5 seconds until the status becomes `SUCCEEDED`.

## 1.4 Transcript Normalization

Once transcription is ready, the workflow merges transcript segments into one continuous text, calculates word count, and estimates duration in minutes.

## 1.5 AI Blog Generation

The cleaned transcript is passed to an AI Agent using GPT-4o-mini. The model is instructed to return only strict JSON for a complete SEO article structure. A structured output parser validates and auto-fixes the AI output when needed.

## 1.6 Storage and Response

The generated SEO fields are appended to Google Sheets as a new draft row, and the workflow returns the resulting payload to the webhook caller.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview

This block exposes the workflow as an HTTP endpoint. It receives a POST request containing the video URL and keeps the request open until the workflow later sends a response through a Respond to Webhook node.

### Nodes Involved

- Receive Video URL

### Node Details

#### Receive Video URL

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for external callers.
- **Configuration choices:**
  - HTTP method: `POST`
  - Response mode: `responseNode`, meaning the workflow must explicitly respond later using a Respond to Webhook node
  - Webhook path: generated UUID-like path
- **Key expressions or variables used:**
  - Downstream nodes reference `{{$json.body.url}}`
- **Input and output connections:**
  - No input node
  - Outputs to `Submit Transcription — Wayin`
- **Version-specific requirements:**
  - Type version `2.1`
- **Edge cases or potential failure types:**
  - Missing `body.url`
  - Invalid or unsupported video URL
  - Caller timeout if downstream processing takes too long
  - Because response mode is deferred, if later nodes fail and no error handling is added, the caller may receive an error or timeout
- **Sub-workflow reference:** None

---

## Block 2 — Transcription Submission

### Overview

This block submits the incoming video URL to Wayin Video’s transcription API and starts an asynchronous transcription job. The returned job ID is then used by the polling block.

### Nodes Involved

- Submit Transcription — Wayin

### Node Details

#### Submit Transcription — Wayin

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to Wayin Video’s transcription endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/transcripts`
  - Request body sent as JSON
  - Custom headers include authorization and API version
- **Key expressions or variables used:**
  - `{{$json.body.url}}` from the webhook payload
  - Body payload:
    - `video_url`: incoming URL
    - `target_lang`: `"en"`
- **Input and output connections:**
  - Input from `Receive Video URL`
  - Output to `Poll Transcription Status`
- **Version-specific requirements:**
  - Type version `4.4`
- **Edge cases or potential failure types:**
  - Invalid bearer token
  - Wrong API version header
  - Unsupported video provider or malformed URL
  - Rate limiting or upstream service outage
  - If Wayin changes response shape, downstream reference to `data.id` may fail
- **Sub-workflow reference:** None

---

## Block 3 — Polling for Completion

### Overview

This block checks the transcription job result endpoint until the transcription status becomes `SUCCEEDED`. It creates a loop using an If node and a Wait node.

### Nodes Involved

- Poll Transcription Status
- Wait 5 Seconds
- Is Transcription Complete?

### Node Details

#### Poll Transcription Status

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the transcription result endpoint using the job ID from the submission response.
- **Configuration choices:**
  - URL dynamically built with the Wayin job ID:
    - `https://wayinvideo-api.wayin.ai/api/v2/transcripts/results/{{ $('Submit Transcription — Wayin').item.json.data.id }}`
  - Headers repeat the Wayin auth and API version
  - Body is configured as JSON, though for a status/result fetch it appears unnecessary
- **Key expressions or variables used:**
  - `{{$('Submit Transcription — Wayin').item.json.data.id}}`
- **Input and output connections:**
  - First input from `Submit Transcription — Wayin`
  - Loopback input from `Is Transcription Complete` false branch
  - Output to `Wait 5 Seconds`
- **Version-specific requirements:**
  - Type version `4.4`
- **Edge cases or potential failure types:**
  - Missing or invalid `data.id`
  - API returns processing, failed, or unexpected status
  - Body content may be ignored by the API, but if the endpoint later rejects unnecessary body content, this could fail
  - No explicit timeout/attempt limit means the loop can continue indefinitely
- **Sub-workflow reference:** None

#### Wait 5 Seconds

- **Type and technical role:** `n8n-nodes-base.wait`  
  Suspends execution between polling attempts.
- **Configuration choices:**
  - No explicit custom delay parameters are visible in the JSON, so it relies on the node’s current default behavior/settings in this workflow context
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Poll Transcription Status`
  - Output to `Is Transcription Complete?`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - If the node is not actually configured to wait 5 seconds in the UI, the displayed name may be misleading
  - Wait nodes depend on n8n execution persistence/resume support
- **Sub-workflow reference:** None

#### Is Transcription Complete?

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on transcription status.
- **Configuration choices:**
  - Condition checks whether `{{$json.data.status}}` equals `SUCCEEDED`
  - True branch continues processing
  - False branch returns to polling
- **Key expressions or variables used:**
  - `{{$json.data.status}}`
- **Input and output connections:**
  - Input from `Wait 5 Seconds`
  - True output to `Process Transcript Data`
  - False output to `Poll Transcription Status`
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - Non-string or missing `data.status`
  - Failed states such as `FAILED`, `CANCELLED`, or API error states are not separately handled; they will continue looping if they do not equal `SUCCEEDED`
  - Infinite polling risk
- **Sub-workflow reference:** None

---

## Block 4 — Transcript Normalization

### Overview

This block converts the transcript segment array into a single text string and computes basic content metrics used later by the AI prompt and Sheets output.

### Nodes Involved

- Process Transcript Data

### Node Details

#### Process Transcript Data

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node.
- **Configuration choices:**
  - Reads the first incoming item
  - Extracts `pollResult.data.transcript`
  - Throws an error if no transcript segments exist
  - Joins all segment text with spaces
  - Calculates:
    - `full_transcript`
    - `word_count`
    - `total_segments`
    - `duration_minutes`
- **Key expressions or variables used:**
  - `const pollResult = $input.first().json`
  - `pollResult.data?.transcript || []`
  - Duration based on last segment’s `end`
- **Input and output connections:**
  - Input from `Is Transcription Complete?` true branch
  - Output to `Generate SEO Blog Post`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Empty transcript throws `Transcript empty!`
  - If segment structure differs, `s.text` or `end` may be undefined
  - Duration assumes transcript segments are ordered chronologically
  - Word count is whitespace-based and may overcount/undercount in edge cases
- **Sub-workflow reference:** None

---

## Block 5 — AI Blog Generation

### Overview

This block sends the transcript to an AI Agent that is instructed to generate a complete SEO blog post in strict JSON. A structured output parser is attached to enforce schema compliance, and a second GPT model is used by the parser for repair or normalization.

### Nodes Involved

- Generate SEO Blog Post
- GPT Model — Blog Generator
- Parse Blog JSON Output
- GPT Model — Output Parser

### Node Details

#### Generate SEO Blog Post

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain AI Agent node that orchestrates prompt execution using a connected language model and output parser.
- **Configuration choices:**
  - Prompt type: defined directly in node
  - Main input text: `{{$json.full_transcript}}`
  - Strong system message instructs the model to:
    - act as an expert SEO writer
    - transform transcript into a blog post
    - return only raw JSON
  - Prompt embeds:
    - Video title from Wayin submission response
    - Transcript word count
    - Duration in minutes
    - Full transcript text
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{$json.full_transcript}}`
  - `{{$json.word_count}}`
  - `{{$json.duration_minutes}}`
  - `{{$('Submit Transcription — Wayin').item.json.data.name}}`
- **Input and output connections:**
  - Main input from `Process Transcript Data`
  - AI language model input from `GPT Model — Blog Generator`
  - AI output parser input from `Parse Blog JSON Output`
  - Main output to `Save to Google Sheets`
- **Version-specific requirements:**
  - Type version `3.1`
  - Requires compatible LangChain nodes installed in the n8n instance
- **Edge cases or potential failure types:**
  - Prompt too large if transcript is very long
  - AI may still return invalid JSON despite instructions
  - Missing `data.name` from Wayin response may reduce prompt quality
  - OpenAI auth, quota, or rate-limit failures
  - Hallucinated or low-quality SEO structure if transcript quality is poor
- **Sub-workflow reference:** None

#### GPT Model — Blog Generator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Primary OpenAI chat model used by the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No special options or built-in tools configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Generate SEO Blog Post` as `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials
  - Model access restrictions
  - Token/context limits for long transcripts
  - Temporary API failures
- **Sub-workflow reference:** None

#### Parse Blog JSON Output

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Validates the AI output against a schema and can auto-fix malformed output.
- **Configuration choices:**
  - `autoFix: true`
  - Schema expects:
    - seo_title
    - meta_description
    - slug
    - focus_keyword
    - secondary_keywords
    - introduction
    - sections[]
    - key_takeaways[]
    - faq[]
    - conclusion
    - read_time
    - tags[]
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - AI language model input from `GPT Model — Output Parser`
  - Connected to `Generate SEO Blog Post` as `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If output is too malformed, auto-fix may still fail
  - Schema validates structure, not content quality
  - Arrays may be structurally valid but semantically weak
- **Sub-workflow reference:** None

#### GPT Model — Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Secondary OpenAI model used by the structured output parser to repair invalid outputs if needed.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Parse Blog JSON Output` as `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Same OpenAI credential and quota risks as the main model
  - Can increase latency and cost
- **Sub-workflow reference:** None

---

## Block 6 — Storage and Response

### Overview

This block appends selected blog fields and workflow metadata into Google Sheets, then returns the workflow output to the original webhook caller.

### Nodes Involved

- Save to Google Sheets
- Return Success Response

### Node Details

#### Save to Google Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to a Google Sheet containing the generated SEO metadata.
- **Configuration choices:**
  - Operation: `append`
  - Target document ID placeholder: `YOUR_GOOGLE_SHEET_DOCUMENT_ID`
  - Target sheet: `gid=0`
  - Mapping mode: explicit field mapping
  - Columns written:
    - Date
    - Video URL
    - SEO Title
    - Slug
    - Focus Keyword
    - Meta Description
    - Secondary Keywords
    - Read Time
    - Tags
    - Word Count
    - Duration (min)
    - Status
  - `Status` is hardcoded to `Draft`
- **Key expressions or variables used:**
  - `{{$now.toFormat('dd MMMM yyyy')}}`
  - `{{$json.output.slug}}`
  - `{{$json.output.tags.map((t, i) => (i + 1) + '. ' + t).join('\n')}}`
  - `{{$json.output.seo_title}}`
  - `{{$('Receive Video URL').item.json.body.url}}`
  - `{{$('Process Transcript Data').item.json.word_count}}`
  - `{{$json.output.focus_keyword}}`
  - `{{$('Process Transcript Data').item.json.duration_minutes}}`
  - `{{$json.output.meta_description}}`
  - `{{$json.output.secondary_keywords.map(k => '• ' + k).join('\n')}}`
  - `{{$json.output.read_time}}`
- **Input and output connections:**
  - Input from `Generate SEO Blog Post`
  - Output to `Return Success Response`
- **Version-specific requirements:**
  - Type version `4.7`
  - Requires Google Sheets OAuth or service-account-style access supported by the node configuration
- **Edge cases or potential failure types:**
  - Invalid or missing Google credentials
  - Wrong sheet ID or inaccessible spreadsheet
  - Column mismatch if the target sheet does not contain the expected headers
  - If `output.tags` or `output.secondary_keywords` are not arrays, expression evaluation will fail
- **Sub-workflow reference:** None

#### Return Success Response

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final response back to the caller of the initial webhook.
- **Configuration choices:**
  - Respond with: `allIncomingItems`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Save to Google Sheets`
  - No output
- **Version-specific requirements:**
  - Type version `1.5`
- **Edge cases or potential failure types:**
  - If upstream nodes fail, this node is never reached
  - Responding with all incoming items may include more data than desired for some API consumers
- **Sub-workflow reference:** None

---

## Additional Non-Executable Nodes — Sticky Notes

These nodes document the workflow visually inside the canvas.

### Nodes Involved

- Sticky Note
- 📋 Workflow Overview
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Notes

These nodes do not affect execution but are important for maintainability. Their content is included in the summary table below, mapped to the nodes they visually describe.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Video URL | n8n-nodes-base.webhook | Accepts incoming POST request with video URL |  | Submit Transcription — Wayin | ## 1. Receive Video URL<br>Webhook listens for a POST request containing the video URL.<br>**Method:** POST<br>**Body:** `{ "url": "https://youtube.com/watch?v=..." }` |
| Submit Transcription — Wayin | n8n-nodes-base.httpRequest | Creates Wayin transcription job | Receive Video URL | Poll Transcription Status | ## 2. Send to Wayin for Transcription<br>Submits the video URL to WayinVideo AI — an accurate multilingual transcription API — and receives a unique job ID in return.<br>🔗 Learn more: https://wayin.ai |
| Poll Transcription Status | n8n-nodes-base.httpRequest | Retrieves transcription job status/result | Submit Transcription — Wayin; Is Transcription Complete? | Wait 5 Seconds | ## 3. Poll Until Transcription Completes<br>Fetches job status every 5 seconds using the job ID.<br>- ✅ Status = SUCCEEDED → move forward<br>- 🔁 Any other status → wait and re-poll |
| Wait 5 Seconds | n8n-nodes-base.wait | Delays between polling attempts | Poll Transcription Status | Is Transcription Complete? | ## 3. Poll Until Transcription Completes<br>Fetches job status every 5 seconds using the job ID.<br>- ✅ Status = SUCCEEDED → move forward<br>- 🔁 Any other status → wait and re-poll |
| Is Transcription Complete? | n8n-nodes-base.if | Routes execution based on status | Wait 5 Seconds | Process Transcript Data; Poll Transcription Status | ## 3. Poll Until Transcription Completes<br>Fetches job status every 5 seconds using the job ID.<br>- ✅ Status = SUCCEEDED → move forward<br>- 🔁 Any other status → wait and re-poll |
| Process Transcript Data | n8n-nodes-base.code | Joins transcript and calculates metrics | Is Transcription Complete? | Generate SEO Blog Post | ## 4. Process Transcript Data<br>Joins all transcript segments into one clean text block.<br>Calculates total word count and video duration in minutes. |
| Generate SEO Blog Post | @n8n/n8n-nodes-langchain.agent | Generates structured SEO blog JSON from transcript | Process Transcript Data | Save to Google Sheets | ## 5. Generate SEO Blog Post with AI<br>AI Agent (GPT-4o-mini) writes a full structured blog post from the transcript:<br>- SEO title, meta description, URL slug<br>- H2 sections with 300+ words each<br>- Key takeaways, FAQs, conclusion with CTA<br>- Read time and tags<br>Structured Output Parser validates and auto-fixes the JSON schema. |
| GPT Model — Blog Generator | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies main LLM for blog generation |  | Generate SEO Blog Post | ## 5. Generate SEO Blog Post with AI<br>AI Agent (GPT-4o-mini) writes a full structured blog post from the transcript:<br>- SEO title, meta description, URL slug<br>- H2 sections with 300+ words each<br>- Key takeaways, FAQs, conclusion with CTA<br>- Read time and tags<br>Structured Output Parser validates and auto-fixes the JSON schema. |
| Parse Blog JSON Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON schema | GPT Model — Output Parser | Generate SEO Blog Post | ## 5. Generate SEO Blog Post with AI<br>AI Agent (GPT-4o-mini) writes a full structured blog post from the transcript:<br>- SEO title, meta description, URL slug<br>- H2 sections with 300+ words each<br>- Key takeaways, FAQs, conclusion with CTA<br>- Read time and tags<br>Structured Output Parser validates and auto-fixes the JSON schema. |
| GPT Model — Output Parser | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies LLM for parser auto-fix/repair |  | Parse Blog JSON Output | ## 5. Generate SEO Blog Post with AI<br>AI Agent (GPT-4o-mini) writes a full structured blog post from the transcript:<br>- SEO title, meta description, URL slug<br>- H2 sections with 300+ words each<br>- Key takeaways, FAQs, conclusion with CTA<br>- Read time and tags<br>Structured Output Parser validates and auto-fixes the JSON schema. |
| Save to Google Sheets | n8n-nodes-base.googleSheets | Appends generated blog draft metadata to sheet | Generate SEO Blog Post | Return Success Response | ## 6. Save Draft & Respond<br>Appends the complete blog post as a new row in Google Sheets.<br>- Status set to: Draft<br>- Includes: date, video URL, word count, read time<br>- Webhook returns success confirmation to the caller |
| Return Success Response | n8n-nodes-base.respondToWebhook | Returns final payload to webhook caller | Save to Google Sheets |  | ## 6. Save Draft & Respond<br>Appends the complete blog post as a new row in Google Sheets.<br>- Status set to: Draft<br>- Includes: date, video URL, word count, read time<br>- Webhook returns success confirmation to the caller |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for webhook input block |  |  | ## 1. Receive Video URL<br>Webhook listens for a POST request containing the video URL.<br>**Method:** POST<br>**Body:** `{ "url": "https://youtube.com/watch?v=..." }` |
| 📋 Workflow Overview | n8n-nodes-base.stickyNote | Global canvas documentation and setup notes |  |  | **Video → SEO Blog Post — Automated Pipeline**<br>Automatically convert any video URL into a fully structured SEO blog post using **Wayin AI transcription** + **GPT-4o-mini**. Paste a video link, get a complete draft in Google Sheets — title, slug, meta, sections, FAQs, tags and more.<br><br>**How it works**<br>1. Webhook receives a video URL via POST request<br>2. Wayin API transcribes the video to English text<br>3. Workflow polls every 5 seconds until transcription succeeds<br>4. JavaScript code joins segments and calculates word count & duration<br>5. AI Agent (GPT-4o-mini) writes a full SEO blog post from the transcript<br>6. Structured Output Parser validates and enforces strict JSON schema<br>7. Final blog draft is saved as a new row in Google Sheets<br><br>**Setup**<br>☐ Add Wayin API key in `Submit Transcription — Wayin` and `Poll Transcription Status` nodes<br>☐ Add OpenAI API credentials to `GPT Model — Blog Generator` and `GPT Model — Output Parser`<br>☐ Connect Google Sheets OAuth in `Save to Google Sheets`<br>☐ Replace `YOUR_GOOGLE_SHEET_DOCUMENT_ID` with your actual Sheet ID<br>☐ Create sheet with columns: `Date` \| `Video URL` \| `SEO Title` \| `Slug` \| `Focus Keyword` \| `Meta Description` \| `Secondary Keywords` \| `Read Time` \| `Tags` \| `Word Count` \| `Duration (min)` \| `Status`<br><br>**Customize**<br>- **Language:** Change `target_lang` in Wayin node to transcribe non-English videos<br>- **AI Model:** Swap `gpt-4o-mini` with `gpt-4o` for higher quality output<br>- **Blog Structure:** Edit system prompt in `Generate SEO Blog Post` to add/remove sections<br>- **Output:** Replace Google Sheets with a WordPress or Notion node to auto-publish |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for transcription submission block |  |  | ## 2. Send to Wayin for Transcription<br>Submits the video URL to WayinVideo AI — an accurate multilingual transcription API — and receives a unique job ID in return.<br>🔗 Learn more: https://wayin.ai |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for polling block |  |  | ## 3. Poll Until Transcription Completes<br>Fetches job status every 5 seconds using the job ID.<br>- ✅ Status = SUCCEEDED → move forward<br>- 🔁 Any other status → wait and re-poll |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for transcript processing block |  |  | ## 4. Process Transcript Data<br>Joins all transcript segments into one clean text block.<br>Calculates total word count and video duration in minutes. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for AI generation block |  |  | ## 5. Generate SEO Blog Post with AI<br>AI Agent (GPT-4o-mini) writes a full structured blog post from the transcript:<br><br>- SEO title, meta description, URL slug<br>- H2 sections with 300+ words each<br>- Key takeaways, FAQs, conclusion with CTA<br>- Read time and tags<br><br>Structured Output Parser validates and auto-fixes the JSON schema. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for final storage block |  |  | ## 6. Save Draft & Respond<br>Appends the complete blog post as a new row in Google Sheets.<br><br>- Status set to: Draft<br>- Includes: date, video URL, word count, read time<br>- Webhook returns success confirmation to the caller |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named `Receive Video URL`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - Set any unique path you want
   - Expected request body:
     ```json
     { "url": "https://youtube.com/watch?v=..." }
     ```

3. **Add an HTTP Request node** named `Submit Transcription — Wayin`.
   - Type: `HTTP Request`
   - Method: `POST`
   - URL: `https://wayinvideo-api.wayin.ai/api/v2/transcripts`
   - Enable headers
   - Add headers:
     - `Authorization`: `Bearer YOUR_TOKEN_HERE`
     - `Content-Type`: `application/json`
     - `x-wayinvideo-api-version`: `v2`
   - Send body as JSON
   - JSON body:
     ```json
     {
       "video_url": "{{$json.body.url}}",
       "target_lang": "en"
     }
     ```
   - Connect `Receive Video URL` → `Submit Transcription — Wayin`

4. **Add an HTTP Request node** named `Poll Transcription Status`.
   - Method: typically GET-like retrieval, but reproduce exactly as configured if needed in your environment
   - URL:
     ```text
     =https://wayinvideo-api.wayin.ai/api/v2/transcripts/results/{{ $('Submit Transcription — Wayin').item.json.data.id }}
     ```
   - Add the same headers:
     - `Authorization`: `Bearer YOUR_TOKEN_HERE`
     - `Content-Type`: `application/json`
     - `x-wayinvideo-api-version`: `v2`
   - The current workflow also includes a JSON body, though it may not be required:
     ```json
     {
       "video_url": "https://www.youtube.com/watch?v=JkJOZewGTU4",
       "target_lang": "en"
     }
     ```
   - Connect `Submit Transcription — Wayin` → `Poll Transcription Status`

5. **Add a Wait node** named `Wait 5 Seconds`.
   - Configure it to pause 5 seconds between polls
   - Connect `Poll Transcription Status` → `Wait 5 Seconds`

6. **Add an If node** named `Is Transcription Complete?`.
   - Condition:
     - Left value: `={{ $json.data.status }}`
     - Operator: `equals`
     - Right value: `SUCCEEDED`
   - Connect `Wait 5 Seconds` → `Is Transcription Complete?`

7. **Create the polling loop.**
   - Connect the **true** output of `Is Transcription Complete?` → `Process Transcript Data`
   - Connect the **false** output of `Is Transcription Complete?` → `Poll Transcription Status`

8. **Add a Code node** named `Process Transcript Data`.
   - Paste this JavaScript logic:
     ```javascript
     const pollResult = $input.first().json;
     const segments = pollResult.data?.transcript || [];

     if (segments.length === 0) {
       throw new Error('Transcript empty!');
     }

     const fullText = segments
       .map(s => s.text.trim())
       .join(' ');

     const wordCount = fullText.split(/\s+/).length;

     const durationMs = segments[segments.length - 1]?.end || 0;
     const durationMin = Math.round(durationMs / 60000);

     return [{
       json: {
         full_transcript: fullText,
         word_count: wordCount,
         total_segments: segments.length,
         duration_minutes: durationMin
       }
     }];
     ```
   - Connect `Is Transcription Complete?` true branch → `Process Transcript Data`

9. **Add an AI Agent node** named `Generate SEO Blog Post`.
   - Type: LangChain Agent
   - Prompt type: define in node
   - Main text input:
     ```text
     ={{ $json.full_transcript }}
     ```
   - Enable output parser
   - Use this system message:
     ```text
     =You are an expert SEO content writer. Transform video transcripts into 
     high-quality SEO blog posts. Return ONLY valid raw JSON — no markdown, 
     no backticks, no explanation.

     Video Title: {{ $('Submit Transcription — Wayin').item.json.data.name }}
     Transcript ({{ $json.word_count }} words | {{ $json.duration_minutes }} min):
     ---
     {{ $json.full_transcript }}
     ---

     Return ONLY this JSON:
     {
       "seo_title": "55-60 char title with keyword",
       "meta_description": "155 char meta with CTA",
       "slug": "url-slug",
       "focus_keyword": "main keyword",
       "secondary_keywords": ["kw2","kw3","kw4"],
       "introduction": "3 hook paragraphs",
       "sections": [{"h2": "heading", "content": "300+ words"}],
       "key_takeaways": ["p1","p2","p3","p4","p5"],
       "faq": [{"question": "Q?", "answer": "answer"}],
       "conclusion": "2 paragraphs with CTA",
       "read_time": "X min read",
       "tags": ["tag1","tag2","tag3"]
     }
     ```
   - Connect `Process Transcript Data` → `Generate SEO Blog Post`

10. **Add an OpenAI Chat Model node** named `GPT Model — Blog Generator`.
    - Provider: OpenAI
    - Model: `gpt-4o-mini`
    - Attach valid OpenAI credentials
    - Connect it to `Generate SEO Blog Post` as the AI language model

11. **Add a Structured Output Parser node** named `Parse Blog JSON Output`.
    - Enable `Auto-fix`
    - Provide this schema example:
      ```json
      {
        "seo_title": "string",
        "meta_description": "string",
        "slug": "string",
        "focus_keyword": "string",
        "secondary_keywords": ["string"],
        "introduction": "string",
        "sections": [
          {
            "h2": "string",
            "content": "string"
          }
        ],
        "key_takeaways": ["string"],
        "faq": [
          {
            "question": "string",
            "answer": "string"
          }
        ],
        "conclusion": "string",
        "read_time": "string",
        "tags": ["string"]
      }
      ```
    - Connect `Parse Blog JSON Output` to `Generate SEO Blog Post` as the output parser

12. **Add another OpenAI Chat Model node** named `GPT Model — Output Parser`.
    - Model: `gpt-4o-mini`
    - Use OpenAI credentials
    - Connect it to `Parse Blog JSON Output` as its language model

13. **Prepare the Google Sheet** before building the Sheets node.
    - Create a spreadsheet
    - Add a worksheet with these headers:
      - `Date`
      - `Video URL`
      - `SEO Title`
      - `Slug`
      - `Focus Keyword`
      - `Meta Description`
      - `Secondary Keywords`
      - `Read Time`
      - `Tags`
      - `Word Count`
      - `Duration (min)`
      - `Status`

14. **Add a Google Sheets node** named `Save to Google Sheets`.
    - Operation: `Append`
    - Connect Google Sheets credentials
    - Select your spreadsheet document ID
    - Select the correct sheet/tab
    - Use explicit column mapping
    - Map fields as follows:
      - `Date` → `={{ $now.toFormat('dd MMMM yyyy') }}`
      - `Video URL` → `={{ $('Receive Video URL').item.json.body.url }}`
      - `SEO Title` → `={{ $json.output.seo_title }}`
      - `Slug` → `={{ $json.output.slug }}`
      - `Focus Keyword` → `={{ $json.output.focus_keyword }}`
      - `Meta Description` → `={{ $json.output.meta_description }}`
      - `Secondary Keywords` → `={{ $json.output.secondary_keywords.map(k => '• ' + k).join('\n') }}`
      - `Read Time` → `={{ $json.output.read_time }}`
      - `Tags` → `={{ $json.output.tags.map((t, i) => (i + 1) + '. ' + t).join('\n') }}`
      - `Word Count` → `={{ $('Process Transcript Data').item.json.word_count }}`
      - `Duration (min)` → `={{ $('Process Transcript Data').item.json.duration_minutes }}`
      - `Status` → `Draft`
    - Connect `Generate SEO Blog Post` → `Save to Google Sheets`

15. **Add a Respond to Webhook node** named `Return Success Response`.
    - Respond with: `All Incoming Items`
    - Connect `Save to Google Sheets` → `Return Success Response`

16. **Add credentials.**
    - **Wayin API**
      - Replace `YOUR_TOKEN_HERE` in both HTTP Request nodes with a valid bearer token
    - **OpenAI**
      - Configure credentials for both OpenAI model nodes
    - **Google Sheets**
      - Connect OAuth2 or the Google auth method supported by your n8n instance

17. **Test the workflow.**
    - Start with the webhook test URL
    - Send a POST request such as:
      ```bash
      curl -X POST 'https://YOUR_N8N/webhook-test/your-path' \
        -H 'Content-Type: application/json' \
        -d '{"url":"https://www.youtube.com/watch?v=JkJOZewGTU4"}'
      ```
    - Confirm:
      - Wayin returns a job ID
      - polling continues until `SUCCEEDED`
      - transcript metrics are generated
      - AI returns structured JSON
      - Google Sheets receives a new row
      - webhook caller receives the final payload

18. **Optional hardening improvements recommended during rebuild.**
    - Add validation for missing `body.url`
    - Add a branch for `FAILED` or `ERROR` transcription statuses
    - Add a max retry count to prevent infinite loops
    - Consider truncating or chunking very long transcripts before sending to the model
    - Consider storing the full article body in Sheets or another destination, because the current mapping saves only selected metadata fields, not the full introduction/sections/FAQ text

### Sub-workflow setup

- This workflow does **not** invoke any sub-workflow.
- It has a single entry point: `Receive Video URL`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Wayin Video is used as the transcription provider. | https://wayin.ai |
| The global canvas note describes the pipeline as “Video → SEO Blog Post — Automated Pipeline”. | Internal workflow note |
| The setup note explicitly asks for replacing `YOUR_GOOGLE_SHEET_DOCUMENT_ID` with a real spreadsheet ID. | Google Sheets configuration |
| The setup note recommends adding OpenAI credentials to both GPT model nodes. | OpenAI configuration |
| The setup note recommends adding the Wayin API key in both HTTP Request nodes. | Wayin API configuration |
| The workflow can be customized by changing `target_lang` for non-English transcription. | Wayin request body |
| The workflow can be customized by replacing `gpt-4o-mini` with `gpt-4o` for higher-quality generation. | OpenAI model selection |
| The workflow can be extended by replacing Google Sheets with WordPress or Notion for publishing. | Destination customization |

## Important implementation observations

- The workflow name says “YouTube videos”, but the webhook accepts a generic `url`, so supported sources depend on Wayin Video API capabilities.
- The polling loop currently treats every non-`SUCCEEDED` status as “keep waiting”. In production, explicit handling for failed jobs is strongly recommended.
- The `Wait 5 Seconds` node is named as if a 5-second delay is configured, but the visible JSON does not show a specific delay parameter. Verify this in the editor.
- The AI output is saved only partially to Google Sheets. The full generated blog structure appears in the workflow output but not in the sheet row unless you add more columns.
- The `Poll Transcription Status` node contains a JSON body that appears unrelated to the polled job and may be unnecessary. This is worth simplifying during maintenance.