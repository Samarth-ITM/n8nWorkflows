Convert YouTube transcripts in Google Sheets to X (Twitter) threads with Apify, OpenAI and RapidAPI

https://n8nworkflows.xyz/workflows/convert-youtube-transcripts-in-google-sheets-to-x--twitter--threads-with-apify--openai-and-rapidapi-14001


# Convert YouTube transcripts in Google Sheets to X (Twitter) threads with Apify, OpenAI and RapidAPI

# 1. Workflow Overview

This workflow automates the conversion of YouTube video transcripts listed in a Google Sheet into X (Twitter) threads. It retrieves one unprocessed YouTube URL from the sheet, scrapes the transcript via Apify, rewrites the transcript into a thread using OpenAI, publishes the thread through a RapidAPI-based X API, and finally marks the source row as processed.

Typical use cases:
- Repurposing long-form YouTube content into short-form social posts
- Running a daily content publishing pipeline from a spreadsheet queue
- Creating a lightweight editorial system where Google Sheets acts as the backlog

## 1.1 Scheduled Input and Runtime Configuration

The workflow starts on a schedule and loads three runtime secrets from a Set node:
- Apify API token
- RapidAPI key
- X/Twitter cookie string

These values are then referenced by downstream HTTP nodes.

## 1.2 Google Sheets Queue Retrieval

The workflow reads rows from a configured Google Sheet containing:
- `Video`
- `Processed`

It filters out rows already marked as processed and limits execution to a single item per run.

## 1.3 Transcript Extraction

The selected YouTube URL is submitted to the Apify YouTube Transcript Scraper actor. The actor returns transcript data and video metadata, which are then used as AI input.

## 1.4 AI Thread Generation

A LangChain AI Agent node sends the transcript and video title to an OpenAI chat model. A structured output parser enforces a JSON schema so the result contains a thread array and some metadata fields.

## 1.5 Thread Publishing and Post-Processing

The generated thread is posted to X using a RapidAPI endpoint authenticated with exported browser cookies. If publication succeeds, the matching Google Sheets row is updated to `Processed = Yes`.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Start and Workflow Configuration

### Overview
This block defines when the workflow runs and centralizes required API/authentication values. It serves as the entry point and configuration source for all external integrations.

### Nodes Involved
- Schedule Trigger
- Workflow Configuration

### Node Details

#### 1. Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a time-based schedule.
- **Configuration choices:**  
  Configured with an interval rule that triggers at hour `9`. In practice, this means the workflow runs once per day at 09:00 based on the instance timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  No input. Outputs to `Workflow Configuration`.
- **Version-specific requirements:** `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  - Timezone misunderstandings between server time and expected local time
  - Workflow inactive status prevents execution
- **Sub-workflow reference:** None.

#### 2. Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores manual configuration values in the execution context.
- **Configuration choices:**  
  Defines three string fields:
  - `apifyApiToken`
  - `rapidApiKey`
  - `twitterCookie`
  
  All are empty in the provided workflow and must be filled before use.
- **Key expressions or variables used:**  
  These fields are later referenced as:
  - `$('Workflow Configuration').item.json.apifyApiToken`
  - `$('Workflow Configuration').item.json.rapidApiKey`
  - `$('Workflow Configuration').item.json.twitterCookie`
- **Input and output connections:**  
  Input from `Schedule Trigger`. Output to `Get YouTube Links from Sheet`.
- **Version-specific requirements:** `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - Empty values will cause downstream authentication failures
  - Invalid cookie/header formatting may fail only at publish time
- **Sub-workflow reference:** None.

---

## Block 2 — Google Sheets Intake and Queue Filtering

### Overview
This block reads the spreadsheet backlog, isolates rows not yet processed, and restricts each workflow run to one YouTube link. It acts as a simple queue mechanism.

### Nodes Involved
- Get YouTube Links from Sheet
- Filter Unprocessed Links
- Get One Link Only

### Node Details

#### 3. Get YouTube Links from Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a specific Google Sheets document and worksheet.
- **Configuration choices:**  
  Uses a Google Sheets OAuth2 credential and targets:
  - Document: `n8n storage`
  - Sheet tab: `Youtbe transcripts to X`
  
  No additional filter conditions are defined here; the node appears to fetch available rows.
- **Key expressions or variables used:** None in parameters beyond the configured sheet selection.
- **Input and output connections:**  
  Input from `Workflow Configuration`. Output to `Filter Unprocessed Links`.
- **Version-specific requirements:** `typeVersion 4.7`, Google Sheets OAuth2 credential required.
- **Edge cases or potential failure types:**  
  - Missing or expired Google OAuth2 credentials
  - Spreadsheet moved, renamed, or permission revoked
  - Unexpected sheet schema, especially if `Video` or `Processed` columns are absent
  - Empty sheet results in no downstream processing
- **Sub-workflow reference:** None.

#### 4. Filter Unprocessed Links
- **Type and technical role:** `n8n-nodes-base.filter`  
  Filters incoming items based on the `Processed` field.
- **Configuration choices:**  
  Keeps only rows where:
  - `Processed` is **not equal** to `Yes`
  
  Expression used:
  - `={{ $json.Processed }}`
- **Key expressions or variables used:**  
  Reads `Processed` from each sheet row.
- **Input and output connections:**  
  Input from `Get YouTube Links from Sheet`. Output to `Get One Link Only`.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Rows with values like `yes`, ` YES `, `True`, or blank are treated as unprocessed
  - If the `Processed` column is missing, filtering behavior may be inconsistent
- **Sub-workflow reference:** None.

#### 5. Get One Link Only
- **Type and technical role:** `n8n-nodes-base.limit`  
  Restricts the number of items passed downstream.
- **Configuration choices:**  
  Uses default limit behavior to pass only one item.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Filter Unprocessed Links`. Output to `Apify YouTube Transcript Scraper`.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - If multiple unprocessed rows exist, only the first retrieved row is processed; order depends on Google Sheets node output order
  - If no rows pass the filter, downstream nodes do not run
- **Sub-workflow reference:** None.

---

## Block 3 — YouTube Transcript Extraction

### Overview
This block sends the selected YouTube URL to Apify’s transcript scraper actor and waits synchronously for the transcript dataset items. It transforms a plain sheet URL into structured transcript input for the AI step.

### Nodes Involved
- Apify YouTube Transcript Scraper

### Node Details

#### 6. Apify YouTube Transcript Scraper
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Apify actor API to execute a transcript scraping task.
- **Configuration choices:**  
  - Method: `POST`
  - Timeout: `120000` ms
  - Body format: JSON
  - Endpoint:
    `https://api.apify.com/v2/acts/supreme_coder~youtube-transcript-scraper/run-sync-get-dataset-items?token=...`
  
  The token is injected from `Workflow Configuration`.
  
  Request body:
  - `outputFormat: "text"`
  - `urls`: array containing one object with the selected `Video` URL
  
  Source expression for the URL:
  - `{{ $('Get YouTube Links from Sheet').item.json.Video }}`
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').item.json.apifyApiToken`
  - `$('Get YouTube Links from Sheet').item.json.Video`
- **Input and output connections:**  
  Input from `Get One Link Only`. Output to `Convert to Twitter Thread`.
- **Version-specific requirements:** `typeVersion 4.4`.
- **Edge cases or potential failure types:**  
  - Invalid or empty Apify token
  - Actor unavailable, rate-limited, or changed
  - Video has no transcript/captions available
  - Regional restrictions or private/deleted video
  - Response shape may differ if Apify actor output changes
  - Timeout on long-running scrape
- **Sub-workflow reference:** None.

---

## Block 4 — AI Conversion to Structured X Thread

### Overview
This block converts the scraped transcript into a concise X thread. It uses an OpenAI chat model connected to an AI Agent node, and a structured output parser ensures the response matches a predictable JSON schema.

### Nodes Involved
- Convert to Twitter Thread
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### 7. Convert to Twitter Thread
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates prompt execution using a connected language model and parser.
- **Configuration choices:**  
  Prompt type is explicitly defined. The user prompt instructs the model to:
  - Convert the transcript into a Twitter thread
  - Avoid wording that sounds like a direct live conversion from YouTube
  - Produce:
    1. A strong hook tweet with numbers/results first
    2. 5–10 insight tweets
    3. A CTA tweet
  - Return structured JSON with a `thread` array
  
  System message defines the assistant as an expert Twitter thread creator and adds tweet-level constraints such as under-280 characters and light emoji usage.
- **Key expressions or variables used:**  
  From Apify output:
  - `{{ $json.videoDetails.title }}`
  - `{{ $json.transcript }}`
- **Input and output connections:**  
  Main input from `Apify YouTube Transcript Scraper`.  
  AI language model input from `OpenAI Chat Model`.  
  AI output parser input from `Structured Output Parser`.  
  Main output to `Publish Thread with RapidAPI/X API`.
- **Version-specific requirements:** `typeVersion 3.1`; requires compatible LangChain nodes in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing `videoDetails.title` or `transcript` in Apify response
  - Model output failing schema validation
  - Hallucinated structure or overlong tweets
  - Empty/low-quality transcript leading to poor thread quality
  - LLM quota, auth, or model availability issues
- **Sub-workflow reference:** None.

#### 8. OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the LLM used by the agent.
- **Configuration choices:**  
  - Model: `gpt-5-mini`
  - No extra options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Connected to `Convert to Twitter Thread` as its `ai_languageModel`.
- **Version-specific requirements:** `typeVersion 1.3`; requires an OpenAI API credential configured in n8n.
- **Edge cases or potential failure types:**  
  - Invalid/expired OpenAI API key
  - Model not available in the account/region
  - Usage limits or rate limits
  - Cost considerations for repeated scheduled runs
- **Sub-workflow reference:** None.

#### 9. Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema on model output.
- **Configuration choices:**  
  Schema requires an object with:
  - `thread`: array of objects, each containing `text` (string)
  - `totalTweets`: number
  - `videoTitle`: string
  - `videoUrl`: string
  
  Note: while the schema expects `videoUrl`, the prompt does not explicitly instruct the model to populate it.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Connected to `Convert to Twitter Thread` as its `ai_outputParser`.
- **Version-specific requirements:** `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  - Validation failure if model omits required fields
  - Type mismatch, e.g. `thread` returned as array of strings instead of objects
  - Incomplete outputs due to token limits
- **Sub-workflow reference:** None.

---

## Block 5 — Thread Publication and Sheet Update

### Overview
This block publishes the generated thread to X through a RapidAPI endpoint, then updates the source spreadsheet row so it is not processed again. It completes the content lifecycle from queue item to published output.

### Nodes Involved
- Publish Thread with RapidAPI/X API
- Mark Link as Processed

### Node Details

#### 10. Publish Thread with RapidAPI/X API
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the generated thread to a third-party X posting API hosted on RapidAPI.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://x-twitter-api2.p.rapidapi.com/x/post-thread`
  - Sends headers and body parameters
  
  Headers:
  - `x-rapidapi-host: x-twitter-api2.p.rapidapi.com`
  - `x-rapidapi-key: {{ $('Workflow Configuration').item.json.rapidApiKey }}`
  
  Body:
  - `cookies: {{ $('Workflow Configuration').item.json.twitterCookie }}`
  - `tweets: {{ $json.output.thread }}`
  
  The thread payload is taken from the AI node output.
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').item.json.rapidApiKey`
  - `$('Workflow Configuration').item.json.twitterCookie`
  - `$json.output.thread`
- **Input and output connections:**  
  Input from `Convert to Twitter Thread`. Output to `Mark Link as Processed`.
- **Version-specific requirements:** `typeVersion 4.4`.
- **Edge cases or potential failure types:**  
  - Invalid RapidAPI key or inactive subscription
  - Invalid/expired X cookies
  - API rejects tweet format
  - Thread item structure may not match expected API shape if it expects strings instead of objects
  - Duplicate content or anti-spam restrictions on X
  - Temporary posting failures or 4xx/5xx API errors
- **Sub-workflow reference:** None.

#### 11. Mark Link as Processed
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the processed status for the source row in the Google Sheet.
- **Configuration choices:**  
  Operation: `update`
  
  Matching column:
  - `Video`
  
  Updated values:
  - `Video = {{ $('Get One Link Only').item.json.Video }}`
  - `Processed = Yes`
  - `row_number = 0`
  
  The node uses define-below mapping mode and matches rows by the `Video` column rather than explicit row number.
- **Key expressions or variables used:**  
  - `$('Get One Link Only').item.json.Video`
- **Input and output connections:**  
  Input from `Publish Thread with RapidAPI/X API`. No downstream nodes.
- **Version-specific requirements:** `typeVersion 4.7`; requires Google Sheets OAuth2 credential.
- **Edge cases or potential failure types:**  
  - If duplicate `Video` values exist, update behavior may affect the wrong row or multiple rows depending on node behavior
  - Permission issues on the spreadsheet
  - Sheet schema drift
  - Setting `row_number = 0` is unnecessary here and could be confusing, though matching is based on `Video`
- **Sub-workflow reference:** None.

---

## Block 6 — Documentation and Embedded Notes

### Overview
These nodes are not executed as part of the data flow. They provide setup instructions, sheet format guidance, integration links, and a summary of the workflow’s intended behavior.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 12. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for sheet structure.
- **Configuration choices:**  
  Contains a sample sheet format with columns `Video` and `Processed`.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 13. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for transcript scraping.
- **Configuration choices:**  
  Explains use of the Apify YouTube transcript scraper and includes a link.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 14. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for credential setup.
- **Configuration choices:**  
  Includes instructions for obtaining:
  - X cookies
  - RapidAPI key
  - Apify API token
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:**  
  The Apify token “Click here” link is empty/incomplete.
- **Sub-workflow reference:** None.

#### 15. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for X posting.
- **Configuration choices:**  
  Explains that the RapidAPI X API posts content using account cookies and mentions a free basic plan.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### 16. Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level workflow summary and embedded prompt reference.
- **Configuration choices:**  
  Summarizes workflow steps, features, and both system/user prompt texts.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily entry point |  | Workflow Configuration | ## Youtube to 𝕏 (Twitter) threads<br>This workflow lets you convert youtube video transcript to twitter thread using Google sheets, Apify and RapidAPI<br>### Workflow summary<br>1. Get unprocessed youtube video link from google sheet<br>2. Use youtube trascript scraper to extract transcript from youtube video<br>3. Feed the transcript to OpenAI model with advanced prompt to convert the transcript to twitter friendly posts/thread<br>4. Post the thread to twitter from your account using your cookies for authentication<br>5. Mark the video as Processed so that it won't be processed again<br>### Features<br>- Fast and cheap (mostly free) APIs<br>- Advanced Prompts to avoid tweets sound like AI converted from youtube<br>- Customizable AI Model<br>### Prompt used<br>**System:** You are an expert Twitter thread creator. Your task is to convert YouTube video transcripts into engaging, viral-worthy Twitter threads. Each tweet must be under 280 characters, punchy, and valuable on its own. Use emojis sparingly for emphasis and include line breaks for readability. Always create a hook tweet, 5-10 insight tweets, and a call-to-action tweet.<br>**User:** Convert this YouTube video transcript into a Twitter thread. Make sure it doesn't sound like it is direct conversion from youtube. For example do not make sentences that indicates you are perforing an activity in presense tense. Create a thread with: 1. A hook tweet that grabs attention. Show results first with numbers 2. 5-10 tweets breaking down key insights 3. A final call-to-action tweet Return as structured JSON with the thread array. Video Title: {{Title}} Transcript: {{Transcript}} |
| Workflow Configuration | n8n-nodes-base.set | Stores runtime API secrets | Schedule Trigger | Get YouTube Links from Sheet | ## Setting up credentials<br>### 𝕏 Cookies<br>Follow these steps to get the cookies:<br>- Install [Cookie-Editor](https://chrome.google.com/webstore/detail/hlkenndednhfkekhgcdicdfddnkalmdm) chrome extension<br>- Login to your 𝕏 account<br>- Click on the Cookie-Editor -> Export -> Header string<br>- Paste the copied cookies into cookie input field<br>### RapidAPI Key<br>- Sign up to [RapidAPI](https://rapidapi.com)<br>- [Click here](https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732) to get api key, Copy from input field **X-RapidAPI-Key**<br>### Apify API Token<br>- Signup to [Apify](https://apify.com?fpr=ve081&fp_sid=n8n)<br>- Click here to get api token (Under section **Personal API tokens**) |
| Get YouTube Links from Sheet | n8n-nodes-base.googleSheets | Reads YouTube queue rows | Workflow Configuration | Filter Unprocessed Links | ## Google sheet template<br>Prepare your google sheet in this format and configure this node to use that sheet.<br>| Video | Processed |<br>|-------|--------|<br>| https://www.youtube.com/watch?v=example1 |No|<br>| https://www.youtube.com/watch?v=example2 | No|<br>| https://www.youtube.com/watch?v=example3 | No |
| Filter Unprocessed Links | n8n-nodes-base.filter | Excludes rows already processed | Get YouTube Links from Sheet | Get One Link Only | ## Google sheet template<br>Prepare your google sheet in this format and configure this node to use that sheet.<br>| Video | Processed |<br>|-------|--------|<br>| https://www.youtube.com/watch?v=example1 |No|<br>| https://www.youtube.com/watch?v=example2 | No|<br>| https://www.youtube.com/watch?v=example3 | No |
| Get One Link Only | n8n-nodes-base.limit | Limits each run to one row | Filter Unprocessed Links | Apify YouTube Transcript Scraper | ## Google sheet template<br>Prepare your google sheet in this format and configure this node to use that sheet.<br>| Video | Processed |<br>|-------|--------|<br>| https://www.youtube.com/watch?v=example1 |No|<br>| https://www.youtube.com/watch?v=example2 | No|<br>| https://www.youtube.com/watch?v=example3 | No |
| Apify YouTube Transcript Scraper | n8n-nodes-base.httpRequest | Scrapes transcript from YouTube via Apify | Get One Link Only | Convert to Twitter Thread | ## Scrape youtube transcripts<br>This node uses [apify youtube transcript scraper](https://apify.com/supreme_coder/youtube-transcript-scraper?fpr=ve081) which is the cheapest transcript scraper available right now.<br>You can click on above link to know more about the apify actor.<br>You can view the scraper input, logs and scraped dataset on apify console once this node is executed. |
| Convert to Twitter Thread | @n8n/n8n-nodes-langchain.agent | Converts transcript into structured thread JSON | Apify YouTube Transcript Scraper; OpenAI Chat Model; Structured Output Parser | Publish Thread with RapidAPI/X API |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM for thread generation |  | Convert to Twitter Thread |  |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON structure on LLM output |  | Convert to Twitter Thread |  |
| Publish Thread with RapidAPI/X API | n8n-nodes-base.httpRequest | Publishes thread to X via RapidAPI | Convert to Twitter Thread | Mark Link as Processed | ## Create 𝕏 (Twitter) Thread<br>This node calls [X-API](https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732) service on RapidAPI to post the converted text content to x.com using your cookies.<br>Subscribe to Basic plan which is free |
| Mark Link as Processed | n8n-nodes-base.googleSheets | Updates processed status in the sheet | Publish Thread with RapidAPI/X API |  | ## Create 𝕏 (Twitter) Thread<br>This node calls [X-API](https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732) service on RapidAPI to post the converted text content to x.com using your cookies.<br>Subscribe to Basic plan which is free |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for sheet format |  |  | ## Google sheet template<br>Prepare your google sheet in this format and configure this node to use that sheet.<br>| Video | Processed |<br>|-------|--------|<br>| https://www.youtube.com/watch?v=example1 |No|<br>| https://www.youtube.com/watch?v=example2 | No|<br>| https://www.youtube.com/watch?v=example3 | No |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for Apify scraping |  |  | ## Scrape youtube transcripts<br>This node uses [apify youtube transcript scraper](https://apify.com/supreme_coder/youtube-transcript-scraper?fpr=ve081) which is the cheapest transcript scraper available right now.<br>You can click on above link to know more about the apify actor.<br>You can view the scraper input, logs and scraped dataset on apify console once this node is executed. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for credentials setup |  |  | ## Setting up credentials<br>### 𝕏 Cookies<br>Follow these steps to get the cookies:<br>- Install [Cookie-Editor](https://chrome.google.com/webstore/detail/hlkenndednhfkekhgcdicdfddnkalmdm) chrome extension<br>- Login to your 𝕏 account<br>- Click on the Cookie-Editor -> Export -> Header string<br>- Paste the copied cookies into cookie input field<br>### RapidAPI Key<br>- Sign up to [RapidAPI](https://rapidapi.com)<br>- [Click here](https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732) to get api key, Copy from input field **X-RapidAPI-Key**<br>### Apify API Token<br>- Signup to [Apify](https://apify.com?fpr=ve081&fp_sid=n8n)<br>- Click here to get api token (Under section **Personal API tokens**) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for X publishing |  |  | ## Create 𝕏 (Twitter) Thread<br>This node calls [X-API](https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732) service on RapidAPI to post the converted text content to x.com using your cookies.<br>Subscribe to Basic plan which is free |
| Sticky Note4 | n8n-nodes-base.stickyNote | High-level workflow and prompt reference |  |  | ## Youtube to 𝕏 (Twitter) threads<br>This workflow lets you convert youtube video transcript to twitter thread using Google sheets, Apify and RapidAPI<br>### Workflow summary<br>1. Get unprocessed youtube video link from google sheet<br>2. Use youtube trascript scraper to extract transcript from youtube video<br>3. Feed the transcript to OpenAI model with advanced prompt to convert the transcript to twitter friendly posts/thread<br>4. Post the thread to twitter from your account using your cookies for authentication<br>5. Mark the video as Processed so that it won't be processed again<br>### Features<br>- Fast and cheap (mostly free) APIs<br>- Advanced Prompts to avoid tweets sound like AI converted from youtube<br>- Customizable AI Model<br>### Prompt used<br>**System:** You are an expert Twitter thread creator. Your task is to convert YouTube video transcripts into engaging, viral-worthy Twitter threads. Each tweet must be under 280 characters, punchy, and valuable on its own. Use emojis sparingly for emphasis and include line breaks for readability. Always create a hook tweet, 5-10 insight tweets, and a call-to-action tweet.<br>**User:** Convert this YouTube video transcript into a Twitter thread. Make sure it doesn't sound like it is direct conversion from youtube. For example do not make sentences that indicates you are perforing an activity in presense tense. Create a thread with: 1. A hook tweet that grabs attention. Show results first with numbers 2. 5-10 tweets breaking down key insights 3. A final call-to-action tweet Return as structured JSON with the thread array. Video Title: {{Title}} Transcript: {{Transcript}} |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `YouTube Transcripts to Twitter Threads from Google Sheets`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run once per day at `09:00`.
   - Confirm the workflow timezone matches your intended posting timezone.

3. **Add a Set node named `Workflow Configuration`**
   - Node type: `Set`
   - Add three string fields:
     - `apifyApiToken`
     - `rapidApiKey`
     - `twitterCookie`
   - Paste the actual values into each field.
   - Connect `Schedule Trigger -> Workflow Configuration`.

4. **Prepare the Google Sheet**
   - Create a spreadsheet with at least one worksheet.
   - Add these headers:
     - `Video`
     - `Processed`
   - Example rows:
     - `https://www.youtube.com/watch?v=... | No`
   - Ensure each YouTube URL is unique if you plan to update rows by `Video`.

5. **Create Google Sheets credentials in n8n**
   - Add a `Google Sheets OAuth2 API` credential.
   - Authenticate the Google account that can access the sheet.
   - Make sure the sheet is editable by that account.

6. **Add a Google Sheets node named `Get YouTube Links from Sheet`**
   - Node type: `Google Sheets`
   - Operation: read rows
   - Select the spreadsheet document.
   - Select the worksheet tab containing the queue.
   - Leave it configured to fetch rows from the table.
   - Connect `Workflow Configuration -> Get YouTube Links from Sheet`.

7. **Add a Filter node named `Filter Unprocessed Links`**
   - Node type: `Filter`
   - Add one string condition:
     - Left value: `={{ $json.Processed }}`
     - Operation: `notEqual`
     - Right value: `Yes`
   - Connect `Get YouTube Links from Sheet -> Filter Unprocessed Links`.

8. **Add a Limit node named `Get One Link Only`**
   - Node type: `Limit`
   - Keep the default behavior to pass only one item.
   - Connect `Filter Unprocessed Links -> Get One Link Only`.

9. **Add an HTTP Request node named `Apify YouTube Transcript Scraper`**
   - Node type: `HTTP Request`
   - Method: `POST`
   - URL:
     - `={{ 'https://api.apify.com/v2/acts/supreme_coder~youtube-transcript-scraper/run-sync-get-dataset-items?token=' + $('Workflow Configuration').item.json.apifyApiToken }}`
   - Enable `Send Body`
   - Body content type: `JSON`
   - JSON body:
     - `outputFormat` = `text`
     - `urls` = array with one object containing:
       - `url` = `{{ $('Get YouTube Links from Sheet').item.json.Video }}`
   - Set timeout to `120000` ms.
   - Connect `Get One Link Only -> Apify YouTube Transcript Scraper`.

10. **Create OpenAI credentials in n8n**
    - Add an `OpenAI API` credential.
    - Use an API key with access to the desired chat model.
    - Confirm that `gpt-5-mini` is available in your account; if not, select another compatible model.

11. **Add an OpenAI Chat Model node named `OpenAI Chat Model`**
    - Node type: `OpenAI Chat Model`
    - Model: `gpt-5-mini`
    - Leave other options at defaults unless you want to tune generation behavior.

12. **Add a Structured Output Parser node named `Structured Output Parser`**
    - Node type: `Structured Output Parser`
    - Schema type: manual
    - Define a schema that expects:
      - `thread`: array of objects with `text` string
      - `totalTweets`: number
      - `videoTitle`: string
      - `videoUrl`: string

13. **Add an AI Agent node named `Convert to Twitter Thread`**
    - Node type: `AI Agent`
    - Prompt type: `Define`
    - User prompt:
      - Instruct the model to convert the YouTube transcript into a Twitter thread
      - Require:
        1. Hook tweet with attention-grabbing numbers/results first
        2. 5–10 insight tweets
        3. CTA tweet
      - Ask for structured JSON with the thread array
      - Insert:
        - `{{ $json.videoDetails.title }}`
        - `{{ $json.transcript }}`
    - System message:
      - Define the model as an expert Twitter thread creator
      - Require under-280-character tweets
      - Keep wording punchy and valuable
      - Use emojis sparingly
    - Enable structured output parsing.
    - Connect:
      - `Apify YouTube Transcript Scraper -> Convert to Twitter Thread`
      - `OpenAI Chat Model -> Convert to Twitter Thread` using the AI language model connection
      - `Structured Output Parser -> Convert to Twitter Thread` using the AI output parser connection

14. **Obtain X/Twitter cookies**
    - Log into the target X account in a browser.
    - Use a cookie export extension such as Cookie-Editor.
    - Export as a header string.
    - Paste that string into `twitterCookie` in the `Workflow Configuration` node.

15. **Get a RapidAPI subscription and API key**
    - Subscribe to the X posting API used in the workflow:
      - `x-twitter-api2.p.rapidapi.com`
    - Copy the `X-RapidAPI-Key` value.
    - Paste it into `rapidApiKey` in the `Workflow Configuration` node.

16. **Add an HTTP Request node named `Publish Thread with RapidAPI/X API`**
    - Node type: `HTTP Request`
    - Method: `POST`
    - URL: `https://x-twitter-api2.p.rapidapi.com/x/post-thread`
    - Enable `Send Headers`
    - Add headers:
      - `x-rapidapi-host` = `x-twitter-api2.p.rapidapi.com`
      - `x-rapidapi-key` = `={{ $('Workflow Configuration').item.json.rapidApiKey }}`
    - Enable `Send Body`
    - Add body parameters:
      - `cookies` = `={{ $('Workflow Configuration').item.json.twitterCookie }}`
      - `tweets` = `={{ $json.output.thread }}`
    - Connect `Convert to Twitter Thread -> Publish Thread with RapidAPI/X API`.

17. **Add a second Google Sheets node named `Mark Link as Processed`**
    - Node type: `Google Sheets`
    - Operation: `Update`
    - Use the same spreadsheet and worksheet as the read node.
    - Set matching column to `Video`.
    - Map fields:
      - `Video` = `={{ $('Get One Link Only').item.json.Video }}`
      - `Processed` = `Yes`
    - You do not need `row_number` unless your sheet setup requires it.
    - Connect `Publish Thread with RapidAPI/X API -> Mark Link as Processed`.

18. **Test the workflow block by block**
    - Run the Google Sheets read step and confirm rows are returned.
    - Test the filter and limit nodes to verify one unprocessed row passes through.
    - Test Apify with a known public YouTube video that has captions.
    - Test the AI output and confirm the parser returns valid JSON.
    - Test posting to X with a low-risk account first.

19. **Validate payload compatibility**
    - Confirm the RapidAPI endpoint accepts the exact structure returned in `thread`.
    - If the API expects an array of strings rather than objects like `{ "text": "..." }`, add a transformation node before publishing to map the thread accordingly.

20. **Activate the workflow**
    - Once all credentials and tests pass, activate the workflow so the schedule trigger runs automatically.

### Important build considerations
- The workflow has **one execution entry point**: `Schedule Trigger`.
- There are **no sub-workflows** and no `Execute Workflow` nodes.
- The spreadsheet acts as the job queue; avoid duplicate `Video` values.
- The AI schema expects `videoUrl`, but the prompt does not strongly enforce it. If needed, add the source URL explicitly to the prompt or populate it with a separate Set/Edit Fields node.
- For reliability, consider adding:
  - an IF node to ensure transcript exists
  - retry logic around Apify and RapidAPI requests
  - error handling branch before marking rows as processed

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheet should contain `Video` and `Processed` columns. | Used as the workflow queue/input source |
| Cookie export is done from a logged-in X browser session using Cookie-Editor, exported as a header string. | https://chrome.google.com/webstore/detail/hlkenndednhfkekhgcdicdfddnkalmdm |
| RapidAPI is used to access the X posting service. | https://rapidapi.com |
| X posting endpoint referenced in notes. | https://rapidapi.com/heycuriouscoder/api/x-twitter-api2/playground/apiendpoint_1a09d2c4-6c90-4bec-98ca-7f325dd3c732 |
| Apify is used for transcript scraping. | https://apify.com?fpr=ve081&fp_sid=n8n |
| Apify actor used by the workflow. | https://apify.com/supreme_coder/youtube-transcript-scraper?fpr=ve081 |
| The sticky note for Apify token retrieval contains an empty “Click here” link in the original workflow notes. | Use Apify account settings / Personal API tokens instead |