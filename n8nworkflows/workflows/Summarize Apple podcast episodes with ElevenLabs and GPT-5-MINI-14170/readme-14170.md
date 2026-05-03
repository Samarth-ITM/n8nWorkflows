Summarize Apple podcast episodes with ElevenLabs and GPT-5-MINI

https://n8nworkflows.xyz/workflows/summarize-apple-podcast-episodes-with-elevenlabs-and-gpt-5-mini-14170


# Summarize Apple podcast episodes with ElevenLabs and GPT-5-MINI

# 1. Workflow Overview

This workflow accepts one or more Apple Podcasts episode URLs through an n8n form, resolves each episode to its podcast RSS feed via the iTunes Lookup API, finds the most likely matching audio enclosure in the RSS XML, sends the MP3 URL to ElevenLabs Speech-to-Text for transcription, summarizes the transcript with GPT-5-MINI, then combines all summaries into a single HTML email sent through Gmail.

Primary use cases:
- Quickly digest podcast episodes without listening end-to-end
- Research competitor or industry podcasts
- Share structured podcast insights internally by email

Logical structure:

## 1.1 Input Reception
The workflow starts with a Form Trigger that asks the user to paste Apple Podcast episode URLs, one per line. A Code node splits the text area content into separate items and extracts the Apple podcast ID from each URL.

## 1.2 Podcast Discovery
For each parsed URL, the workflow queries the iTunes Lookup API to retrieve the podcast RSS feed URL. It then fetches the RSS feed and searches the XML for the episode whose title best matches the slug found in the Apple Podcast URL, extracting the MP3 enclosure URL.

## 1.3 Transcription and AI Summarization
The MP3 URL is sent directly to ElevenLabs Scribe using a multipart/form-data HTTP request. The returned transcript is then passed to the OpenAI node, which prompts GPT-5-MINI to produce a tightly structured Markdown summary.

## 1.4 Email Assembly and Delivery
All generated summaries are collected, converted from lightweight Markdown into styled HTML, wrapped into a single email body, and sent through Gmail to a configured recipient.

## 1.5 Documentation / In-Canvas Guidance
Several sticky notes describe the workflow purpose, setup steps, and functional sections directly in the canvas. These are important for manual operators and should be preserved when recreating the workflow.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block collects raw Apple Podcast episode URLs from the user and transforms them into one workflow item per URL. It also extracts a podcast ID from each URL so the iTunes Lookup API can be queried downstream.

### Nodes Involved
- Submit Podcast URLs
- Parse Input URLs

### Node Details

#### Submit Podcast URLs
- **Type and technical role:** `n8n-nodes-base.formTrigger`; entry-point node that exposes a hosted form and starts the workflow.
- **Configuration choices:**
  - Form title: `Apple Podcasts Summarizer`
  - One required textarea field labeled `Input Apple Podcast links`
  - Placeholder shows an Apple Podcast episode URL example
  - Attribution footer disabled
- **Key expressions or variables used:**
  - Produces a field named exactly: `Input Apple Podcast links`
- **Input and output connections:**
  - Input: none, this is the workflow trigger
  - Output: `Parse Input URLs`
- **Version-specific requirements:**
  - Uses node type version `2.3`
  - Requires n8n instance support for Form Trigger nodes
- **Edge cases or potential failure types:**
  - Empty form submission is prevented by required field setting
  - If users paste malformed or non-Apple URLs, downstream extraction may fail silently
  - Multiple URLs must be separated by line breaks; comma-separated input will not split correctly
- **Sub-workflow reference:** none

#### Parse Input URLs
- **Type and technical role:** `n8n-nodes-base.code`; parses the textarea into discrete items and extracts a numeric podcast ID.
- **Configuration choices:**
  - Reads the textarea field from the first input item
  - Splits text using newline characters
  - Filters out blank lines
  - Uses regex `id(\d+)` to extract the podcast ID from each URL
- **Key expressions or variables used:**
  - `const input = $input.first().json['Input Apple Podcast links'];`
  - `url.match(/id(\d+)/)`
  - Output fields:
    - `apple_url`
    - `podcastId`
- **Input and output connections:**
  - Input: `Submit Podcast URLs`
  - Output: `Look Up RSS Feed via iTunes`
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If the field label changes, the code breaks because it references the exact label
  - Some Apple Podcast URLs may not contain `id123...` in the expected pattern
  - A blank or invalid match yields `podcastId: ''`, causing likely lookup failure downstream
  - Does not validate URL format before continuing
- **Sub-workflow reference:** none

---

## 2.2 Podcast Discovery

### Overview
This block resolves the podcast RSS feed from the Apple podcast ID, downloads the RSS XML, then heuristically identifies the matching episode audio file based on the Apple URL slug. Its output is the episode MP3 URL and a detected episode title.

### Nodes Involved
- Look Up RSS Feed via iTunes
- Extract RSS Feed URL
- Fetch RSS Feed
- Find Episode MP3 URL

### Node Details

#### Look Up RSS Feed via iTunes
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs an HTTP GET request to Apple’s iTunes Lookup API.
- **Configuration choices:**
  - URL expression:
    `https://itunes.apple.com/lookup?id={{ $json.podcastId }}&entity=podcast`
  - Default GET method
  - No custom options configured
- **Key expressions or variables used:**
  - `{{$json.podcastId}}`
- **Input and output connections:**
  - Input: `Parse Input URLs`
  - Output: `Extract RSS Feed URL`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Empty `podcastId` can return invalid or empty results
  - Apple API may rate-limit or temporarily fail
  - Response schema assumptions may break if Apple changes the API
- **Sub-workflow reference:** none

#### Extract RSS Feed URL
- **Type and technical role:** `n8n-nodes-base.code`; normalizes the iTunes API response and extracts `feedUrl`.
- **Configuration choices:**
  - Runs once for each item
  - Handles either:
    - `rawData.data` containing a stringified payload
    - `rawData.data` containing an object
    - direct JSON response
  - Extracts `data.results[0].feedUrl`
- **Key expressions or variables used:**
  - `rawData.data`
  - `JSON.parse(...)`
  - Output field: `rssUrl`
- **Input and output connections:**
  - Input: `Look Up RSS Feed via iTunes`
  - Output: `Fetch RSS Feed`
- **Version-specific requirements:**
  - Code node version `2`
  - Mode: `runOnceForEachItem`
- **Edge cases or potential failure types:**
  - If the HTTP Request node returns a different structure than expected, parsing may fail
  - `results` may be empty, producing `rssUrl: ''`
  - Invalid JSON string in `data` would throw a parse error
- **Sub-workflow reference:** none

#### Fetch RSS Feed
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the podcast RSS feed XML.
- **Configuration choices:**
  - URL expression: `={{ $json.rssUrl }}`
  - Default request settings
- **Key expressions or variables used:**
  - `{{$json.rssUrl}}`
- **Input and output connections:**
  - Input: `Extract RSS Feed URL`
  - Output: `Find Episode MP3 URL`
- **Version-specific requirements:**
  - HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Empty RSS URL causes request failure
  - Feed may be inaccessible, redirected, blocked, or malformed
  - Very large feeds may increase execution time
- **Sub-workflow reference:** none

#### Find Episode MP3 URL
- **Type and technical role:** `n8n-nodes-base.code`; parses RSS XML as text and selects the best matching episode enclosure URL.
- **Configuration choices:**
  - Runs once for each item
  - Reads RSS payload from `$input.item.json.data`
  - Retrieves original Apple URL from `$('Parse Input URLs').item.json.apple_url`
  - Extracts a probable slug from the URL path
  - Converts slug into search terms
  - Splits XML by `<item>`
  - Scores candidate items by counting slug-word matches in episode titles
  - Extracts the first matching audio enclosure URL
  - Falls back to the first audio enclosure in the feed if no match is found
- **Key expressions or variables used:**
  - `$('Parse Input URLs').item.json.apple_url`
  - Output fields:
    - `mp3Url`
    - `episodeTitle`
    - `matchScore`
    - `success`
- **Input and output connections:**
  - Input: `Fetch RSS Feed`
  - Output: `Transcribe Episode with ElevenLabs`
- **Version-specific requirements:**
  - Code node version `2`
  - Mode: `runOnceForEachItem`
- **Edge cases or potential failure types:**
  - XML is parsed by regex/string operations, not a true XML parser; feed formatting differences may break matching
  - URL slug detection is heuristic and may fail for short, unusual, or localized Apple URLs
  - If no title matches, it uses the first episode audio file, which may summarize the wrong episode
  - Enclosure extraction assumes `type="audio"` and double quotes
  - Cross-item references rely on item linking behavior; modifying execution order or node behavior could affect this
- **Sub-workflow reference:** none

---

## 2.3 Transcription and AI Summarization

### Overview
This block transcribes each detected MP3 through ElevenLabs Scribe, then summarizes the transcript with GPT-5-MINI using a strict Markdown template. The prompt is designed to minimize hallucination and enforce consistent email-ready structure.

### Nodes Involved
- Transcribe Episode with ElevenLabs
- Generate Episode Summary

### Node Details

#### Transcribe Episode with ElevenLabs
- **Type and technical role:** `n8n-nodes-base.httpRequest`; POST request to ElevenLabs Speech-to-Text API using multipart form data.
- **Configuration choices:**
  - URL: `https://api.elevenlabs.io/v1/speech-to-text`
  - Method: `POST`
  - Body type: `multipart-form-data`
  - Generic credential type: `httpHeaderAuth`
  - Multipart fields:
    - `cloud_storage_url` = `{{$json.mp3Url}}`
    - `model_id` = `scribe_v2`
  - Timeout: `600000` ms (10 minutes)
- **Key expressions or variables used:**
  - `{{$json.mp3Url}}`
- **Input and output connections:**
  - Input: `Find Episode MP3 URL`
  - Output: `Generate Episode Summary`
- **Version-specific requirements:**
  - HTTP Request node version `4.2`
  - Requires an HTTP Header Auth credential containing the ElevenLabs API key
- **Edge cases or potential failure types:**
  - Invalid or inaccessible MP3 URL will fail transcription
  - Audio file may be too large, unsupported, expired, or blocked from remote fetch
  - Long episodes can exceed timeout or provider processing constraints
  - Authentication failures if API key is missing or invalid
  - Response format may vary depending on ElevenLabs API version
- **Sub-workflow reference:** none

#### Generate Episode Summary
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; sends transcript text to OpenAI for structured summarization.
- **Configuration choices:**
  - Model: `gpt-5-mini`
  - `simplify` disabled, preserving fuller response shape
  - Single prompt message containing:
    - strict formatting rules
    - explicit word limits
    - prefilled Title and URL fields
    - transcript injected from `{{$json.text}}`
  - The prompt instructs the model to output only Markdown matching the template
- **Key expressions or variables used:**
  - `{{ $('Find Episode MP3 URL').item.json.episodeTitle }}`
  - `{{ $('Parse Input URLs').item.json.apple_url }}`
  - `{{ $json.text }}`
- **Input and output connections:**
  - Input: `Transcribe Episode with ElevenLabs`
  - Output: `Build HTML Email`
- **Version-specific requirements:**
  - OpenAI node version `1.8`
  - Requires a configured OpenAI API credential
  - Availability of `gpt-5-mini` depends on account/model access in the connected OpenAI project
- **Edge cases or potential failure types:**
  - If transcription output does not contain `text`, the prompt will be incomplete
  - Large transcripts may exceed model token limits or increase cost/latency
  - Cross-node item references must stay aligned per item
  - Model may still occasionally deviate from formatting instructions
  - Credential, quota, or model-access errors can stop the workflow
- **Sub-workflow reference:** none

---

## 2.4 Email Assembly and Delivery

### Overview
This block gathers all summary outputs, converts Markdown-like content into HTML, composes a consolidated email body, and sends it to the configured Gmail recipient.

### Nodes Involved
- Build HTML Email
- Send Summary Email

### Node Details

#### Build HTML Email
- **Type and technical role:** `n8n-nodes-base.code`; aggregates all summary items and renders them into a styled HTML email.
- **Configuration choices:**
  - Uses `$input.all()` to collect every summary item
  - Includes helper function `mdToHtml(md)` that:
    - converts `- ` lines into `<ul><li>`
    - converts `**bold**` into `<strong>`
    - converts Markdown links into HTML `<a>`
    - wraps non-list lines in `<p>`
  - Builds an HTML document with:
    - title `Podcast Summaries`
    - generation date
    - episode count
    - one bordered section per summary
  - Tries multiple response shapes to extract summary text:
    - `item.message.content`
    - `item.choices[0].message.content`
    - `item.content`
    - fallback error string
- **Key expressions or variables used:**
  - `$input.all()`
  - Output field: `html`
- **Input and output connections:**
  - Input: `Generate Episode Summary`
  - Output: `Send Summary Email`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - If OpenAI response shape changes, extraction may fail and produce fallback text
  - Markdown conversion is intentionally simple and may not support all formatting
  - Summary text containing unexpected HTML-sensitive characters could affect rendering
  - If no items arrive, the email may still send with an empty body section
- **Sub-workflow reference:** none

#### Send Summary Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the assembled HTML email using Gmail OAuth2.
- **Configuration choices:**
  - Recipient: `user@example.com`
  - Subject: `Podcast Summary`
  - Message/body: `={{ $json.html }}`
  - Attribution disabled
  - `executeOnce: false` but receives only the single aggregated HTML item from the previous node
- **Key expressions or variables used:**
  - `{{$json.html}}`
- **Input and output connections:**
  - Input: `Build HTML Email`
  - Output: none
- **Version-specific requirements:**
  - Gmail node version `2.1`
  - Requires Gmail OAuth2 credentials with permission to send mail
- **Edge cases or potential failure types:**
  - Recipient is a placeholder and must be changed before real use
  - Gmail OAuth token expiration or scope mismatch can block sending
  - Some email clients may render the generated HTML differently
  - Gmail sending quotas or anti-abuse checks may apply
- **Sub-workflow reference:** none

---

## 2.5 Documentation / In-Canvas Guidance

### Overview
These sticky notes document the workflow’s purpose, setup requirements, and the role of each section. They do not execute logic, but they are operationally useful and should be preserved.

### Nodes Involved
- Overview
- Section: User Input
- Section: Podcast Discovery
- Section: Transcription & Summary
- Section: Email Delivery

### Node Details

#### Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:**
  - Large note containing purpose, audience, flow description, and setup checklist
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none in execution
- **Sub-workflow reference:** none

#### Section: User Input
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the input block.
- **Configuration choices:**
  - Text: `① User Input`
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Section: Podcast Discovery
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the RSS/feed resolution block.
- **Configuration choices:**
  - Text: `② Podcast Discovery`
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Section: Transcription & Summary
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the transcription and summarization block.
- **Configuration choices:**
  - Text: `③ Transcription & AI Summary`
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Section: Email Delivery
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the email output block.
- **Configuration choices:**
  - Text: `④ Email Delivery`
- **Key expressions or variables used:** none
- **Input and output connections:** none
- **Version-specific requirements:** sticky note version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | Canvas documentation with workflow purpose, setup, and operating notes |  |  | ## 🎙️ Summarize Apple Podcast Episodes with ElevenLabs and GPT-5-MINI<br><br>**Paste one or more Apple Podcast episode URLs into a form (one per line) and receive a structured AI-generated summary by email - powered by ElevenLabs speech-to-text and GPT-5-MINI.**<br><br>---<br><br>### Who is this for<br>- Podcast listeners who want fast episode digests<br>- Content creators researching competitor podcasts<br>- Teams that share podcast insights via email<br><br>---<br><br>### How it works<br>1. **Submit URLs** - Paste one or more Apple Podcast episode URLs into the trigger form<br>2. **Discover RSS feed** - The iTunes API is queried to find the podcast's public RSS feed URL<br>3. **Find episode MP3** - The RSS XML is parsed to locate the matching episode audio file<br>4. **Transcribe audio** - ElevenLabs Scribe transcribes the full episode via direct URL<br>5. **Generate summary** - GPT-5-MINI produces a structured summary: title, key points, useful info, and bottom line<br>6. **Email results** - All summaries are combined into a formatted HTML email and sent to your inbox<br><br>---<br><br>### Setup<br>1. **ElevenLabs** - Add HTTP Header Auth credential with your ElevenLabs API key; connect to "Transcribe Episode with ElevenLabs"<br>2. **OpenAI** - Connect your OpenAI API credential to "Generate Episode Summary"<br>3. **Gmail** - Connect your Gmail OAuth2 credential to "Send Summary Email"<br>4. **Recipient email** - Update the "To" address in "Send Summary Email"<br>5. Activate the workflow |
| Section: User Input | n8n-nodes-base.stickyNote | Canvas label for the input section |  |  | ## ① User Input<br>User submits Apple Podcast episode URLs via the n8n form. |
| Section: Podcast Discovery | n8n-nodes-base.stickyNote | Canvas label for RSS and episode discovery |  |  | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Section: Transcription & Summary | n8n-nodes-base.stickyNote | Canvas label for transcription and summarization |  |  | ## ③ Transcription & AI Summary<br>Transcribes the episode audio and generates a structured summary with GPT-5-MINI. |
| Section: Email Delivery | n8n-nodes-base.stickyNote | Canvas label for email assembly and send |  |  | ## ④ Email Delivery<br>Combines all summaries into an HTML email and sends it to the recipient. |
| Submit Podcast URLs | n8n-nodes-base.formTrigger | Workflow entry form for one or more Apple Podcast URLs |  | Parse Input URLs | ## ① User Input<br>User submits Apple Podcast episode URLs via the n8n form. |
| Parse Input URLs | n8n-nodes-base.code | Split textarea content into items and extract podcast IDs | Submit Podcast URLs | Look Up RSS Feed via iTunes | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Look Up RSS Feed via iTunes | n8n-nodes-base.httpRequest | Query iTunes Lookup API for the podcast feed URL | Parse Input URLs | Extract RSS Feed URL | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Extract RSS Feed URL | n8n-nodes-base.code | Parse iTunes lookup response and output the RSS feed URL | Look Up RSS Feed via iTunes | Fetch RSS Feed | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Fetch RSS Feed | n8n-nodes-base.httpRequest | Download the RSS XML feed for the podcast | Extract RSS Feed URL | Find Episode MP3 URL | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Find Episode MP3 URL | n8n-nodes-base.code | Heuristically match the requested Apple episode to an RSS enclosure URL | Fetch RSS Feed | Transcribe Episode with ElevenLabs | ## ② Podcast Discovery<br>Looks up the RSS feed via iTunes API and extracts the matching episode MP3 URL(s). |
| Transcribe Episode with ElevenLabs | n8n-nodes-base.httpRequest | Submit remote MP3 URL to ElevenLabs Speech-to-Text | Find Episode MP3 URL | Generate Episode Summary | ## ③ Transcription & AI Summary<br>Transcribes the episode audio and generates a structured summary with GPT-5-MINI. |
| Generate Episode Summary | @n8n/n8n-nodes-langchain.openAi | Generate a structured Markdown summary from the transcript | Transcribe Episode with ElevenLabs | Build HTML Email | ## ③ Transcription & AI Summary<br>Transcribes the episode audio and generates a structured summary with GPT-5-MINI. |
| Build HTML Email | n8n-nodes-base.code | Aggregate all summaries and convert them into a styled HTML email | Generate Episode Summary | Send Summary Email | ## ④ Email Delivery<br>Combines all summaries into an HTML email and sends it to the recipient. |
| Send Summary Email | n8n-nodes-base.gmail | Send the final HTML summary email through Gmail | Build HTML Email |  | ## ④ Email Delivery<br>Combines all summaries into an HTML email and sends it to the recipient. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Summarize Apple Podcast Episodes with ElevenLabs and GPT-5-MINI`.

2. **Add the Form Trigger node**
   - Node type: `Form Trigger`
   - Name: `Submit Podcast URLs`
   - Form title: `Apple Podcasts Summarizer`
   - Add one field:
     - Type: `Textarea`
     - Label: `Input Apple Podcast links`
     - Required: enabled
     - Placeholder: any Apple Podcast episode URL example
   - In options, disable attribution if desired.
   - This is the workflow entry point.

3. **Add a Code node after the form**
   - Node type: `Code`
   - Name: `Parse Input URLs`
   - Connect: `Submit Podcast URLs` → `Parse Input URLs`
   - Paste this logic conceptually:
     - Read `Input Apple Podcast links`
     - Split on line breaks
     - Remove empty lines
     - For each URL:
       - trim whitespace
       - extract `podcastId` using regex matching `id` followed by digits
       - output `apple_url` and `podcastId`
   - Important: keep the form field label exactly `Input Apple Podcast links` unless you also update the code.

4. **Add an HTTP Request node for iTunes Lookup**
   - Node type: `HTTP Request`
   - Name: `Look Up RSS Feed via iTunes`
   - Connect: `Parse Input URLs` → `Look Up RSS Feed via iTunes`
   - Method: `GET`
   - URL:
     `https://itunes.apple.com/lookup?id={{ $json.podcastId }}&entity=podcast`
   - Leave other settings default.

5. **Add a Code node to extract the feed URL**
   - Node type: `Code`
   - Name: `Extract RSS Feed URL`
   - Connect: `Look Up RSS Feed via iTunes` → `Extract RSS Feed URL`
   - Set mode: `Run Once for Each Item`
   - Logic:
     - Read the lookup response
     - If response is inside `data`, normalize it
     - If needed, parse JSON string into object
     - Extract `results[0].feedUrl`
     - Return `{ rssUrl }`

6. **Add an HTTP Request node to fetch the RSS feed**
   - Node type: `HTTP Request`
   - Name: `Fetch RSS Feed`
   - Connect: `Extract RSS Feed URL` → `Fetch RSS Feed`
   - Method: `GET`
   - URL:
     `{{ $json.rssUrl }}`
   - Default settings are sufficient.

7. **Add a Code node to identify the episode MP3**
   - Node type: `Code`
   - Name: `Find Episode MP3 URL`
   - Connect: `Fetch RSS Feed` → `Find Episode MP3 URL`
   - Set mode: `Run Once for Each Item`
   - Implement logic that:
     - Reads the RSS XML as text
     - Reads the original Apple URL from `Parse Input URLs`
     - Derives a slug-like set of search terms from the Apple URL path
     - Splits the XML into `<item>` sections
     - Extracts each item title
     - Scores items based on how many slug words appear in the title
     - Selects the best match
     - Extracts the audio enclosure URL
     - Falls back to the first audio enclosure if no match is found
   - Return fields:
     - `mp3Url`
     - `episodeTitle`
     - `matchScore`
     - `success`

8. **Create ElevenLabs credentials**
   - Credential type: `HTTP Header Auth`
   - Name it something like `ElevenLabs API`
   - Configure the required header expected by ElevenLabs, typically their API key header
   - Save the credential.

9. **Add the ElevenLabs transcription request**
   - Node type: `HTTP Request`
   - Name: `Transcribe Episode with ElevenLabs`
   - Connect: `Find Episode MP3 URL` → `Transcribe Episode with ElevenLabs`
   - Method: `POST`
   - URL: `https://api.elevenlabs.io/v1/speech-to-text`
   - Authentication: `Generic Credential Type`
   - Generic auth type: `HTTP Header Auth`
   - Select the ElevenLabs credential
   - Enable body sending
   - Content type: `Multipart Form-Data`
   - Add body parameters:
     - `cloud_storage_url` = `{{ $json.mp3Url }}`
     - `model_id` = `scribe_v2`
   - Set timeout to `600000` ms

10. **Create OpenAI credentials**
    - Credential type: `OpenAI API`
    - Name it something like `OpenAi account`
    - Enter your API key
    - Ensure the account has access to `gpt-5-mini`

11. **Add the OpenAI summarization node**
    - Node type: `OpenAI` using the LangChain/OpenAI integration node
    - Name: `Generate Episode Summary`
    - Connect: `Transcribe Episode with ElevenLabs` → `Generate Episode Summary`
    - Select model: `gpt-5-mini`
    - Disable simplify output
    - Add one message prompt instructing the model to:
      - summarize only from transcript content
      - preserve exact Markdown structure
      - keep the title and URL unchanged
      - produce sections:
        - `**Title**`
        - `**URL**`
        - `**Main Topic:**`
        - `**Key Points:**`
        - `**Useful Info:**`
        - `**Bottom Line:**`
    - Inject these expressions:
      - Title from `Find Episode MP3 URL` → `episodeTitle`
      - URL from `Parse Input URLs` → `apple_url`
      - Transcript from current item → `text`
    - Keep the prompt strict, because downstream HTML conversion assumes predictable Markdown.

12. **Add a Code node to build the email**
    - Node type: `Code`
    - Name: `Build HTML Email`
    - Connect: `Generate Episode Summary` → `Build HTML Email`
    - Logic should:
      - collect all incoming summary items with `$input.all()`
      - attempt to extract summary text from likely OpenAI output fields
      - convert simple Markdown formatting into HTML:
        - bullet lists
        - bold text
        - links
      - build an HTML wrapper with:
        - heading `Podcast Summaries`
        - generation date
        - count of episodes
        - one styled block per summary
      - output one field: `html`

13. **Create Gmail credentials**
    - Credential type: `Gmail OAuth2`
    - Name it something like `Gmail account`
    - Authorize the account with send-mail permissions

14. **Add the Gmail node**
    - Node type: `Gmail`
    - Name: `Send Summary Email`
    - Connect: `Build HTML Email` → `Send Summary Email`
    - Operation: send email
    - To: replace placeholder with your actual recipient, for example `you@yourdomain.com`
    - Subject: `Podcast Summary`
    - Message/body:
      `{{ $json.html }}`
    - Disable attribution if desired

15. **Add canvas notes for maintainability**
    - Add sticky notes for:
      - overall overview and setup
      - user input
      - podcast discovery
      - transcription and AI summary
      - email delivery
    - This is optional for execution, but recommended for human operators.

16. **Test with one Apple Podcast episode URL**
    - Submit a known public Apple episode URL through the form
    - Confirm these outputs step-by-step:
      - `podcastId` is extracted
      - iTunes returns a valid feed URL
      - RSS fetch succeeds
      - MP3 URL is found
      - ElevenLabs returns transcript text
      - OpenAI returns formatted Markdown
      - Email contains rendered HTML summary

17. **Test with multiple URLs**
    - Paste several episode URLs, one per line
    - Confirm the workflow processes each as a separate item
    - Confirm only one final email is sent containing all summaries

18. **Activate the workflow**
    - Ensure all credentials are connected
    - Replace placeholder email recipient
    - Activate the workflow so the form endpoint is live

### Credential summary
- **ElevenLabs:** HTTP Header Auth credential for the Speech-to-Text API
- **OpenAI:** OpenAI API credential with access to `gpt-5-mini`
- **Gmail:** Gmail OAuth2 credential with send permissions

### Input/output expectations
- **Input:** One or more Apple Podcast episode URLs, one per line
- **Intermediate outputs:** `podcastId`, `rssUrl`, `mp3Url`, `episodeTitle`, transcript `text`
- **Final output:** One HTML email containing all summaries

### Sub-workflow setup
- No sub-workflows are used in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Paste one or more Apple Podcast episode URLs into the form, one per line, to receive structured AI-generated summaries by email. | Workflow purpose |
| Setup requires three credentials: ElevenLabs via HTTP Header Auth, OpenAI API, and Gmail OAuth2. | Operational setup |
| Update the recipient address in `Send Summary Email` before production use. | Configuration reminder |
| Activate the workflow after credentials are connected so the form trigger becomes usable. | Deployment reminder |
| The workflow has an error workflow configured internally: `Nin8EYMkR9vuyKPp`. This referenced workflow is not included here, so its behavior cannot be documented further. | Environment/workflow setting |
| The workflow is currently inactive (`active: false`) in the provided definition. | Deployment state |