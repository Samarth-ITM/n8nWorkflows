Create daily historical AI videos with Gemini, fal.ai, Telegram and YouTube

https://n8nworkflows.xyz/workflows/create-daily-historical-ai-videos-with-gemini--fal-ai--telegram-and-youtube-14372


# Create daily historical AI videos with Gemini, fal.ai, Telegram and YouTube

# 1. Workflow Overview

This workflow automatically creates a daily AI-generated historical video, sends it to Telegram for human approval, and uploads it to YouTube if approved. If the reviewer declines the result, the workflow retries with another historical event, up to a maximum of 3 attempts.

It is designed for automated content pipelines where a human remains in the approval loop before publication. Typical use cases include daily history channels, educational short-form content, and semi-automated YouTube publishing systems.

## 1.1 Daily Trigger and State Initialization
The workflow starts on a daily schedule at 1 AM. It initializes retry state and decides whether to continue or stop based on the retry limit.

## 1.2 Historical Event Retrieval and Script Generation
The workflow fetches historical events for the current calendar date, selects one event at random, and uses Google Gemini to transform it into a cinematic short-form video prompt.

## 1.3 AI Video Generation and Polling
The generated script is submitted to fal.ai for text-to-video generation. The workflow then polls the job status every 30 seconds until the generation is complete.

## 1.4 Video Retrieval and Approval Preparation
Once the video is ready, the workflow retrieves the final video URL, extracts metadata, downloads the file, and prepares a Telegram approval message containing the event context and script excerpt.

## 1.5 Human Approval via Telegram
The generated video is sent to Telegram, followed by an approval step where the user can approve or reject publication.

## 1.6 YouTube Upload and Final Confirmation
If approved, the workflow uploads the video to YouTube and sends a Telegram confirmation message containing the YouTube link.

## 1.7 Retry Handling on Rejection
If the video is rejected, the workflow increments a retry counter and loops back to event selection. Once the maximum retry count is reached, it sends a stop message and terminates.

---

# 2. Block-by-Block Analysis

## 2.1 Daily Trigger and State Initialization

### Overview
This block launches the workflow once per day and manages retry state across loops. It also handles graceful termination when the retry limit has been exceeded.

### Nodes Involved
- Schedule (Daily at 1 AM)
- Initialize Workflow State
- Check Retry Limit
- Send Stop Message

### Node Details

#### Schedule (Daily at 1 AM)
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger.
- **Configuration choices:** Configured to run daily at hour 1.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Initialize Workflow State`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or failure types:** Timezone differences may affect the actual firing time depending on instance settings.
- **Sub-workflow reference:** None.

#### Initialize Workflow State
- **Type and role:** `n8n-nodes-base.code`; initializes or preserves retry state.
- **Configuration choices:** Reads incoming JSON if present. If `retryCount` exists, it preserves it; otherwise starts at 0.
- **Key expressions or variables used:**
  - `$input.first()`
  - `input.json.retryCount`
  - Returns:
    - `retryCount`
    - `attemptNumber`
    - `stopWorkflow`
    - `initialized`
    - `message`
- **Input and output connections:** Input from `Schedule (Daily at 1 AM)`; output to `Check Retry Limit`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:** If reused in a different loop topology, unexpected input shape could cause state confusion, though the script is defensive.
- **Sub-workflow reference:** None.

#### Check Retry Limit
- **Type and role:** `n8n-nodes-base.if`; routes execution depending on whether the workflow should stop.
- **Configuration choices:** Checks whether `{{$json.stopWorkflow}} == true` using loose validation.
- **Key expressions or variables used:**
  - `={{ $json.stopWorkflow }}`
- **Input and output connections:** Input from `Initialize Workflow State` and also from `Increment Retry Counter`; true branch goes to `Send Stop Message`, false branch goes to `Fetch Historical Events`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failure types:** Uses loose type validation, which is helpful if booleans are sometimes represented as strings.
- **Sub-workflow reference:** None.

#### Send Stop Message
- **Type and role:** `n8n-nodes-base.telegram`; notifies the reviewer that the retry limit was reached.
- **Configuration choices:** Sends a text message to a fixed Telegram chat ID using the `message` field from upstream data.
- **Key expressions or variables used:**
  - `={{ $json.message }}`
  - Hardcoded `chatId`: `123456789`
- **Input and output connections:** Input from `Check Retry Limit` true branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or failure types:** Telegram auth failure, invalid chat ID, bot not allowed to message the chat.
- **Sub-workflow reference:** None.

---

## 2.2 Historical Event Retrieval and Script Generation

### Overview
This block retrieves historical events for the current date, randomly selects one, and transforms it into a cinematic text-to-video prompt using Gemini.

### Nodes Involved
- Fetch Historical Events
- Select Random Historical Event
- Generate Cinematic Script with AI
- Gemini Language Model

### Node Details

#### Fetch Historical Events
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches daily historical data from the public Muffinlabs History API.
- **Configuration choices:** Uses a dynamic URL based on the current month and day; timeout set to 30 seconds.
- **Key expressions or variables used:**
  - `=https://history.muffinlabs.com/date/{{ new Date().getMonth() + 1 }}/{{ new Date().getDate() }}`
- **Input and output connections:** Input from `Check Retry Limit` false branch; output to `Select Random Historical Event`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or failure types:** API downtime, rate limiting, unexpected response shape, timeout.
- **Sub-workflow reference:** None.

#### Select Random Historical Event
- **Type and role:** `n8n-nodes-base.code`; chooses a random event and cleans its description.
- **Configuration choices:** Reads `data.Events`, selects one randomly, removes HTML entities, and propagates retry state.
- **Key expressions or variables used:**
  - `$input.first().json.data`
  - `events[Math.floor(Math.random() * events.length)]`
  - `retryCount`
  - `attemptNumber`
- **Input and output connections:** Input from `Fetch Historical Events`; output to `Generate Cinematic Script with AI`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:**
  - Missing `data.Events`
  - Empty event list
  - Unexpected API formatting
- **Sub-workflow reference:** None.

#### Generate Cinematic Script with AI
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; AI agent node used to produce a concise cinematic video prompt.
- **Configuration choices:** Prompt is defined directly in the node. It instructs the model to return a single clean paragraph under 400 words with scene description, mood, camera movement, and visual style.
- **Key expressions or variables used:**
  - `{{ $json.description }}`
  - `{{ $json.year }}`
  - System message: “You are a professional scriptwriter specializing in cinematic historical documentaries.”
- **Input and output connections:** Main input from `Select Random Historical Event`; AI language model input from `Gemini Language Model`; main output to `Submit to fal.ai`.
- **Version-specific requirements:** Type version `3.1`; requires LangChain-compatible n8n AI nodes.
- **Edge cases or failure types:** LLM credential issues, model quota exhaustion, unexpected output formatting, empty result.
- **Sub-workflow reference:** None.

#### Gemini Language Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; supplies the underlying Google Gemini model to the agent node.
- **Configuration choices:** Minimal configuration; relies on linked credentials.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to `Generate Cinematic Script with AI` via `ai_languageModel`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** Invalid Google Gemini credentials, API quota issues, unsupported region/model availability.
- **Sub-workflow reference:** None.

---

## 2.3 AI Video Generation and Polling

### Overview
This block submits the generated prompt to fal.ai and repeatedly checks the generation status until the video is ready.

### Nodes Involved
- Submit to fal.ai
- Check Video Generation Status
- Is Generation Complete?
- Wait 30 Seconds Before Retry
- Get Generated Video

### Node Details

#### Submit to fal.ai
- **Type and role:** `n8n-nodes-base.httpRequest`; starts the video generation job on fal.ai.
- **Configuration choices:** Sends a POST request with JSON body to the `fal-ai/hunyuan-video-lora` queue endpoint using header-based auth.
- **Key expressions or variables used:**
  - Body:
    - `prompt`: `{{ $json.output.replace(/\"/g, '\\"') }}`
    - `aspect_ratio`: `16:9`
    - `resolution`: `720p`
    - `num_frames`: `129`
    - `enable_safety_checker`: `true`
- **Input and output connections:** Input from `Generate Cinematic Script with AI`; output to `Check Video Generation Status`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or failure types:**
  - Invalid API key/header auth
  - fal.ai queue errors
  - malformed JSON body if output contains unusual characters
  - moderation/safety rejection
- **Sub-workflow reference:** None.

#### Check Video Generation Status
- **Type and role:** `n8n-nodes-base.httpRequest`; polls fal.ai for the current status of the request.
- **Configuration choices:** GET request to the `/status` endpoint using `request_id` returned by fal.ai.
- **Key expressions or variables used:**
  - `=https://queue.fal.run/fal-ai/hunyuan-video-lora/requests/{{ $json.request_id }}/status`
- **Input and output connections:** Input from `Submit to fal.ai` and from `Wait 30 Seconds Before Retry`; output to `Is Generation Complete?`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or failure types:**
  - Missing `request_id`
  - API auth failure
  - network timeout
  - non-terminal states not equal to `COMPLETED`
- **Sub-workflow reference:** None.

#### Is Generation Complete?
- **Type and role:** `n8n-nodes-base.if`; checks whether fal.ai has completed the generation.
- **Configuration choices:** Compares `{{$json.status}}` to the string `COMPLETED`.
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:** Input from `Check Video Generation Status`; true branch to `Get Generated Video`; false branch to `Wait 30 Seconds Before Retry`.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failure types:** If fal.ai returns `FAILED`, `CANCELLED`, or another terminal state, this workflow treats it as “not completed” and will continue looping indefinitely unless manually stopped.
- **Sub-workflow reference:** None.

#### Wait 30 Seconds Before Retry
- **Type and role:** `n8n-nodes-base.wait`; pauses before polling again.
- **Configuration choices:** Waits 30 seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Is Generation Complete?` false branch; output back to `Check Video Generation Status`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or failure types:** Long-running executions may consume workflow execution capacity depending on hosting mode.
- **Sub-workflow reference:** None.

#### Get Generated Video
- **Type and role:** `n8n-nodes-base.httpRequest`; fetches the completed fal.ai job payload containing the final video metadata.
- **Configuration choices:** GET request to the request endpoint using the same `request_id`.
- **Key expressions or variables used:**
  - `=https://queue.fal.run/fal-ai/hunyuan-video-lora/requests/{{ $json.request_id }}`
- **Input and output connections:** Input from `Is Generation Complete?` true branch; output to `Extract Video Metadata`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or failure types:** Missing `request_id`, incomplete payload, temporary API inconsistency after completion.
- **Sub-workflow reference:** None.

---

## 2.4 Video Retrieval and Approval Preparation

### Overview
This block extracts the generated video URL and related metadata, downloads the binary file, and creates an approval caption for Telegram.

### Nodes Involved
- Extract Video Metadata
- Download Video File
- Prepare Approval Message
- Send Video for Approval

### Node Details

#### Extract Video Metadata
- **Type and role:** `n8n-nodes-base.code`; normalizes fal.ai output and merges it with event/script context.
- **Configuration choices:** Reads `response.video.url`, fetches event data from `Select Random Historical Event`, and fetches the generated script from `Generate Cinematic Script with AI`.
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$node["Select Random Historical Event"].json`
  - `$node["Generate Cinematic Script with AI"].json.output`
  - Throws error if no video URL is found
- **Input and output connections:** Input from `Get Generated Video`; output to `Download Video File`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:**
  - `video.url` missing
  - dependency on cross-node references makes this node sensitive to renaming or output schema changes
- **Sub-workflow reference:** None.

#### Download Video File
- **Type and role:** `n8n-nodes-base.httpRequest`; downloads the generated video as a file.
- **Configuration choices:** Uses `videoUrl` from upstream; response format is set to file; timeout is 300000 ms.
- **Key expressions or variables used:**
  - `={{ $json.videoUrl }}`
- **Input and output connections:** Input from `Extract Video Metadata`; output to `Prepare Approval Message`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or failure types:**
  - expired or inaccessible video URL
  - download timeout
  - large file issues
- **Sub-workflow reference:** None.

#### Prepare Approval Message
- **Type and role:** `n8n-nodes-base.code`; creates caption text and forwards contextual metadata.
- **Configuration choices:** Builds a Telegram-formatted message including event description, year, script excerpt, and retry attempt information.
- **Key expressions or variables used:**
  - `$node["Extract Video Metadata"].json`
  - `substring(0, 150)`
  - `substring(0, 200)`
  - `attemptNumber`
- **Input and output connections:** Input from `Download Video File`; output to `Send Video for Approval`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:** Since it uses `$node[...]` references, upstream node rename or output changes will break it. Telegram Markdown formatting may also fail if content contains reserved characters.
- **Sub-workflow reference:** None.

#### Send Video for Approval
- **Type and role:** `n8n-nodes-base.telegram`; sends the generated video to the reviewer.
- **Configuration choices:** Uses Telegram `sendVideo` operation and a fixed chat ID. Additional fields are empty in this export.
- **Key expressions or variables used:**
  - Hardcoded `chatId`: `123456789`
- **Input and output connections:** Input from `Prepare Approval Message`; output to `Wait for User Approval`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or failure types:**
  - The node appears not to explicitly map binary file or caption fields in the exported parameters. In practice, this may require manual adjustment after import to ensure the downloaded file and caption are actually sent.
  - Telegram bot permission limits
  - invalid chat ID
- **Sub-workflow reference:** None.

---

## 2.5 Human Approval via Telegram

### Overview
This block asks the user to approve or reject the generated video and branches the workflow based on the response.

### Nodes Involved
- Wait for User Approval
- Check User Approval

### Node Details

#### Wait for User Approval
- **Type and role:** `n8n-nodes-base.telegram`; sends an approval request and pauses until the user responds.
- **Configuration choices:** Uses `sendAndWait` with double approval mode. Message is a simple prompt: “Please approve or reject this video:”.
- **Key expressions or variables used:**
  - Hardcoded `chatId`: `123456789`
  - Approval mode: `double`
- **Input and output connections:** Input from `Send Video for Approval`; output to `Check User Approval`.
- **Version-specific requirements:** Type version `1.2`; requires Telegram interaction features available in the installed n8n version.
- **Edge cases or failure types:**
  - bot webhook/setup issues
  - user never responds, causing long waiting executions
  - wrong chat ID or inaccessible chat
- **Sub-workflow reference:** None.

#### Check User Approval
- **Type and role:** `n8n-nodes-base.if`; determines whether the video should be uploaded or retried.
- **Configuration choices:** Checks whether `{{$json.data.approved}} == true`.
- **Key expressions or variables used:**
  - `={{ $json.data.approved }}`
- **Input and output connections:** Input from `Wait for User Approval`; true branch to `Upload to YouTube`; false branch to `Increment Retry Counter`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or failure types:** If Telegram response payload shape changes, `data.approved` may not exist. String/boolean mismatches can also affect branching.
- **Sub-workflow reference:** None.

---

## 2.6 YouTube Upload and Final Confirmation

### Overview
If the reviewer approves the video, this block uploads it to YouTube and sends a confirmation message back to Telegram.

### Nodes Involved
- Upload to YouTube
- Upload Confirmation Message
- Send Upload Confirmation

### Node Details

#### Upload to YouTube
- **Type and role:** `n8n-nodes-base.youTube`; uploads the approved video to YouTube.
- **Configuration choices:** Upload operation on resource `video`, region `US`. The title is derived from the event description and year.
- **Key expressions or variables used:**
  - `={{ $node['Extract Video Metadata'].json.eventDescription.substring(0, 100) }} ({{ $node['Extract Video Metadata'].json.eventYear }})`
- **Input and output connections:** Input from `Check User Approval` true branch; output to `Upload Confirmation Message`.
- **Version-specific requirements:** Type version `1`; requires YouTube OAuth2 credentials with upload permissions.
- **Edge cases or failure types:**
  - The exported node does not visibly show binary property mapping; manual confirmation is needed to ensure the downloaded file is passed correctly for upload.
  - OAuth token expiry or missing scopes
  - title validation issues
  - YouTube quota restrictions
- **Sub-workflow reference:** None.

#### Upload Confirmation Message
- **Type and role:** `n8n-nodes-base.code`; creates a final Telegram message containing the YouTube watch URL.
- **Configuration choices:** Reads the uploaded video ID from input and event details from `Extract Video Metadata`.
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$node["Extract Video Metadata"].json`
  - `https://www.youtube.com/watch?v=${uploadResult.id}`
- **Input and output connections:** Input from `Upload to YouTube`; output to `Send Upload Confirmation`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:** If YouTube upload response does not contain `id`, the message will be invalid.
- **Sub-workflow reference:** None.

#### Send Upload Confirmation
- **Type and role:** `n8n-nodes-base.telegram`; notifies the user that upload succeeded.
- **Configuration choices:** Sends `{{$json.message}}` to the same hardcoded Telegram chat ID.
- **Key expressions or variables used:**
  - `={{ $json.message }}`
  - Hardcoded `chatId`: `123456789`
- **Input and output connections:** Input from `Upload Confirmation Message`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or failure types:** Telegram auth/chat errors.
- **Sub-workflow reference:** None.

---

## 2.7 Retry Handling on Rejection

### Overview
If the user rejects the generated video, this block increments the retry counter and either loops back for another event or stops the workflow once the limit is reached.

### Nodes Involved
- Increment Retry Counter
- Check Retry Limit

### Node Details

#### Increment Retry Counter
- **Type and role:** `n8n-nodes-base.code`; increments the retry count and enforces the max retry rule.
- **Configuration choices:** Uses `maxRetries = 3`. If the next retry count is greater than or equal to the limit, it returns `stopWorkflow: true`; otherwise it returns updated retry metadata.
- **Key expressions or variables used:**
  - `$input.first().json.retryCount || 0`
  - `maxRetries = 3`
  - Returns:
    - `stopWorkflow`
    - `message`
    - `retryCount`
    - `attemptNumber`
- **Input and output connections:** Input from `Check User Approval` false branch; output to `Check Retry Limit`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or failure types:** If `retryCount` is not preserved from the earlier path, retries may always restart from zero. In this workflow, the intended logic exists, but data continuity across the approval branch should be tested carefully.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule (Daily at 1 AM) | Schedule Trigger | Starts the workflow daily at 1 AM |  | Initialize Workflow State | **Schedule and state initialization** Handles the daily trigger, retry logic, and workflow termination. |
| Initialize Workflow State | Code | Initializes retry state and workflow metadata | Schedule (Daily at 1 AM) | Check Retry Limit | **Schedule and state initialization** Handles the daily trigger, retry logic, and workflow termination. |
| Check Retry Limit | If | Stops execution or continues to event generation | Initialize Workflow State; Increment Retry Counter | Send Stop Message; Fetch Historical Events | **Schedule and state initialization** Handles the daily trigger, retry logic, and workflow termination. |
| Send Stop Message | Telegram | Sends stop notification after retry exhaustion | Check Retry Limit |  | **Schedule and state initialization** Handles the daily trigger, retry logic, and workflow termination. |
| Fetch Historical Events | HTTP Request | Retrieves historical events for the current date | Check Retry Limit | Select Random Historical Event | **Fetch and process historical event** Retrieves daily history and generates a cinematic script via AI. |
| Select Random Historical Event | Code | Randomly selects and cleans one historical event | Fetch Historical Events | Generate Cinematic Script with AI | **Fetch and process historical event** Retrieves daily history and generates a cinematic script via AI. |
| Generate Cinematic Script with AI | LangChain Agent | Generates a cinematic text-to-video prompt | Select Random Historical Event; Gemini Language Model | Submit to fal.ai | **Fetch and process historical event** Retrieves daily history and generates a cinematic script via AI. |
| Gemini Language Model | Google Gemini Chat Model | Provides the LLM backend for script generation |  | Generate Cinematic Script with AI | **Fetch and process historical event** Retrieves daily history and generates a cinematic script via AI. |
| Submit to fal.ai | HTTP Request | Starts video generation job | Generate Cinematic Script with AI | Check Video Generation Status | **Generate and download AI video** Submits script to fal.ai and monitors generation progress. |
| Check Video Generation Status | HTTP Request | Polls fal.ai generation status | Submit to fal.ai; Wait 30 Seconds Before Retry | Is Generation Complete? | **Generate and download AI video** Submits script to fal.ai and monitors generation progress. |
| Is Generation Complete? | If | Checks whether fal.ai job is complete | Check Video Generation Status | Get Generated Video; Wait 30 Seconds Before Retry | **Generate and download AI video** Submits script to fal.ai and monitors generation progress. |
| Wait 30 Seconds Before Retry | Wait | Delays before the next polling cycle | Is Generation Complete? | Check Video Generation Status | **Generate and download AI video** Submits script to fal.ai and monitors generation progress. |
| Get Generated Video | HTTP Request | Retrieves completed video generation payload | Is Generation Complete? | Extract Video Metadata | **Generate and download AI video** Submits script to fal.ai and monitors generation progress. |
| Extract Video Metadata | Code | Extracts video URL and merges event/script metadata | Get Generated Video | Download Video File | **Retrieve and prepare video file** Fetches the final file and extracts metadata for the approval message. |
| Download Video File | HTTP Request | Downloads generated video as binary file | Extract Video Metadata | Prepare Approval Message | **Retrieve and prepare video file** Fetches the final file and extracts metadata for the approval message. |
| Prepare Approval Message | Code | Builds Telegram approval caption and metadata | Download Video File | Send Video for Approval | **Retrieve and prepare video file** Fetches the final file and extracts metadata for the approval message. |
| Send Video for Approval | Telegram | Sends the generated video to Telegram | Prepare Approval Message | Wait for User Approval | **Retrieve and prepare video file** Fetches the final file and extracts metadata for the approval message. |
| Wait for User Approval | Telegram | Waits for reviewer approval/rejection | Send Video for Approval | Check User Approval |  |
| Check User Approval | If | Branches on approval status | Wait for User Approval | Upload to YouTube; Increment Retry Counter |  |
| Increment Retry Counter | Code | Increments retry count after rejection | Check User Approval | Check Retry Limit | **Retry Handling on Decline** Increments retry counter (max 3) and loops back to select a new event. |
| Upload to YouTube | YouTube | Uploads approved video to YouTube | Check User Approval | Upload Confirmation Message | **YouTube Upload & Confirmation** Uploads approved video to YouTube and sends final confirmation with link via Telegram. |
| Upload Confirmation Message | Code | Builds final upload success message | Upload to YouTube | Send Upload Confirmation | **YouTube Upload & Confirmation** Uploads approved video to YouTube and sends final confirmation with link via Telegram. |
| Send Upload Confirmation | Telegram | Sends YouTube link confirmation to Telegram | Upload Confirmation Message |  | **YouTube Upload & Confirmation** Uploads approved video to YouTube and sends final confirmation with link via Telegram. |
| Event Selection | Sticky Note | Documentation note |  |  |  |
| Video Generation | Sticky Note | Documentation note |  |  |  |
| User Approval | Sticky Note | Documentation note |  |  |  |
| Retry on Decline | Sticky Note | Documentation note |  |  |  |
| YouTube Upload | Sticky Note | Documentation note |  |  |  |
| Daily Schedule & Init | Sticky Note | Documentation note |  |  |  |
| Main Workflow Overview | Sticky Note | Global workflow documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Name it `Schedule (Daily at 1 AM)`
   - Set it to run daily at hour `1`
   - Confirm your workflow timezone in n8n settings

3. **Add a Code node for state initialization**
   - Name: `Initialize Workflow State`
   - Connect from the schedule trigger
   - Paste logic that:
     - reads `$input.first()`
     - preserves `retryCount` if present
     - returns:
       - `retryCount`
       - `attemptNumber`
       - `stopWorkflow: false`
       - `initialized: true`
       - `message`

4. **Add an If node to evaluate workflow stop condition**
   - Name: `Check Retry Limit`
   - Connect from `Initialize Workflow State`
   - Condition:
     - Left value: `={{ $json.stopWorkflow }}`
     - Operator: `equals`
     - Right value: `=true`
   - Use loose validation if needed

5. **Add a Telegram node for stop notifications**
   - Name: `Send Stop Message`
   - Connect to the **true** output of `Check Retry Limit`
   - Operation: send message
   - Chat ID: your Telegram chat ID
   - Text: `={{ $json.message }}`
   - Configure **Telegram credentials**
     - Create a bot with BotFather
     - Add the bot to the target chat if required
     - Use the Telegram API credential in n8n

6. **Add an HTTP Request node to fetch historical events**
   - Name: `Fetch Historical Events`
   - Connect to the **false** output of `Check Retry Limit`
   - Method: GET
   - URL:
     - `=https://history.muffinlabs.com/date/{{ new Date().getMonth() + 1 }}/{{ new Date().getDate() }}`
   - Timeout: `30000`

7. **Add a Code node to select a random event**
   - Name: `Select Random Historical Event`
   - Connect from `Fetch Historical Events`
   - Add code that:
     - reads `data.Events`
     - picks a random event
     - cleans HTML entities from `text`
     - preserves `retryCount`
     - returns:
       - `year`
       - `description`
       - `retryCount`
       - `attemptNumber`

8. **Add a Google Gemini chat model node**
   - Type: `Google Gemini Chat Model`
   - Name: `Gemini Language Model`
   - Create/configure Google Gemini credentials
   - Minimal settings are sufficient if your n8n version matches the exported node family

9. **Add an AI Agent node**
   - Type: `LangChain Agent`
   - Name: `Generate Cinematic Script with AI`
   - Connect main input from `Select Random Historical Event`
   - Connect `Gemini Language Model` to its `ai_languageModel` port
   - Use a defined prompt instructing the model to:
     - act as a cinematic historical documentary scriptwriter
     - transform the event into one paragraph
     - include scene description, mood, camera movement, and visual style
     - stay under 400 words
     - avoid markdown and labels

10. **Add an HTTP Request node to submit the prompt to fal.ai**
    - Name: `Submit to fal.ai`
    - Connect from `Generate Cinematic Script with AI`
    - Method: `POST`
    - URL: `https://queue.fal.run/fal-ai/hunyuan-video-lora`
    - Authentication: `Generic Credential Type` → `HTTP Header Auth`
    - Create HTTP Header Auth credentials for fal.ai
      - Typically include your API key in the expected Authorization header
    - Body type: JSON
    - JSON body:
      - `prompt`: output from Gemini
      - `aspect_ratio`: `16:9`
      - `resolution`: `720p`
      - `num_frames`: `129`
      - `enable_safety_checker`: `true`
    - Escape quotes in the prompt if needed

11. **Add an HTTP Request node for polling status**
    - Name: `Check Video Generation Status`
    - Connect from `Submit to fal.ai`
    - Method: GET
    - URL:
      - `=https://queue.fal.run/fal-ai/hunyuan-video-lora/requests/{{ $json.request_id }}/status`
    - Use the same fal.ai header-auth credential

12. **Add an If node to test completion**
    - Name: `Is Generation Complete?`
    - Connect from `Check Video Generation Status`
    - Condition:
      - Left value: `={{ $json.status }}`
      - Operator: `equals`
      - Right value: `=COMPLETED`

13. **Add a Wait node**
    - Name: `Wait 30 Seconds Before Retry`
    - Connect from the **false** branch of `Is Generation Complete?`
    - Wait amount: `30 seconds`
    - Connect this node back to `Check Video Generation Status`

14. **Add an HTTP Request node to retrieve the completed result**
    - Name: `Get Generated Video`
    - Connect from the **true** branch of `Is Generation Complete?`
    - Method: GET
    - URL:
      - `=https://queue.fal.run/fal-ai/hunyuan-video-lora/requests/{{ $json.request_id }}`
    - Use the same fal.ai auth

15. **Add a Code node to extract metadata**
    - Name: `Extract Video Metadata`
    - Connect from `Get Generated Video`
    - Logic should:
      - read `video.url` from the fal.ai response
      - throw an error if missing
      - read event info from `Select Random Historical Event`
      - read script from `Generate Cinematic Script with AI`
      - return:
        - `videoUrl`
        - `eventYear`
        - `eventDescription`
        - `script`
        - `retryCount`
        - `attemptNumber`

16. **Add an HTTP Request node to download the video file**
    - Name: `Download Video File`
    - Connect from `Extract Video Metadata`
    - URL: `={{ $json.videoUrl }}`
    - Response format: `File`
    - Timeout: `300000`
    - Make note of the binary property name produced by this node, since later Telegram and YouTube upload steps may need it explicitly

17. **Add a Code node to prepare the approval text**
    - Name: `Prepare Approval Message`
    - Connect from `Download Video File`
    - Logic should:
      - read metadata from `Extract Video Metadata`
      - compose a Telegram caption with:
        - event description
        - year
        - script excerpt
        - retry attempt
        - approval instructions
      - return:
        - `caption`
        - `videoUrl`
        - `eventYear`
        - `eventDescription`
        - `script`
        - `attemptNumber`

18. **Add a Telegram node to send the video**
    - Name: `Send Video for Approval`
    - Connect from `Prepare Approval Message`
    - Operation: `sendVideo`
    - Set chat ID to your target chat
    - Configure the node to send the binary video file from `Download Video File`
    - Map the caption from `={{ $json.caption }}`
    - Important: after import/export, verify that the binary property and caption fields are correctly mapped, because they are not explicit in the provided JSON

19. **Add a Telegram node for human approval**
    - Name: `Wait for User Approval`
    - Connect from `Send Video for Approval`
    - Operation: `sendAndWait`
    - Chat ID: same Telegram chat
    - Message: `Please approve or reject this video:`
    - Approval type: `double`
    - Ensure your n8n Telegram integration supports interactive approval waiting

20. **Add an If node to evaluate the approval**
    - Name: `Check User Approval`
    - Connect from `Wait for User Approval`
    - Condition:
      - Left value: `={{ $json.data.approved }}`
      - Operator: `equals`
      - Right value: `=true`

21. **Add a YouTube node**
    - Name: `Upload to YouTube`
    - Connect from the **true** branch of `Check User Approval`
    - Resource: `video`
    - Operation: `upload`
    - Region code: `US`
    - Title:
      - event description truncated to 100 chars plus event year
    - Configure YouTube OAuth2 credentials with upload permission
    - Explicitly map the binary property from `Download Video File`
    - Verify the node has all required metadata fields such as file source and binary property name

22. **Add a Code node to create the upload confirmation**
    - Name: `Upload Confirmation Message`
    - Connect from `Upload to YouTube`
    - Logic should:
      - read uploaded video ID
      - read event info from `Extract Video Metadata`
      - return a message containing the YouTube watch URL

23. **Add a Telegram node for final success notification**
    - Name: `Send Upload Confirmation`
    - Connect from `Upload Confirmation Message`
    - Operation: send message
    - Chat ID: your Telegram chat
    - Text: `={{ $json.message }}`

24. **Add a Code node for retry handling**
    - Name: `Increment Retry Counter`
    - Connect from the **false** branch of `Check User Approval`
    - Logic:
      - read current `retryCount`
      - increment it
      - if `next >= 3`, return:
        - `stopWorkflow: true`
        - stop message
        - updated counts
      - else return:
        - `stopWorkflow: false`
        - updated retry metadata
        - retry message

25. **Connect retry logic back into the stop/continue gate**
    - Connect `Increment Retry Counter` to `Check Retry Limit`
    - This reuses the same branch logic:
      - stop if max retries reached
      - otherwise continue to `Fetch Historical Events`

26. **Add optional sticky notes**
    - Add grouped notes for:
      - schedule/state initialization
      - event selection
      - video generation
      - approval preparation
      - retry handling
      - YouTube upload

27. **Test the workflow in segments**
    - First test event retrieval and Gemini generation
    - Then test fal.ai submission and polling
    - Then validate video download
    - Then verify Telegram file sending and approval behavior
    - Finally confirm YouTube binary upload works

28. **Replace hardcoded chat IDs**
    - The exported workflow hardcodes `123456789`
    - Prefer using:
      - environment variables
      - a Set node
      - workflow static data
    - This makes maintenance and multi-user deployment easier

29. **Activate the workflow**
    - Once credentials and binary mappings are validated, activate the workflow for daily execution

### Credential Setup Required
- **Telegram API**
  - Needed for `Send Stop Message`, `Send Video for Approval`, `Wait for User Approval`, `Send Upload Confirmation`
- **Google Gemini / PaLM API**
  - Needed for `Gemini Language Model`
- **HTTP Header Auth for fal.ai**
  - Needed for `Submit to fal.ai`, `Check Video Generation Status`, `Get Generated Video`
- **YouTube OAuth2**
  - Needed for `Upload to YouTube`

### Important Rebuild Notes
- The workflow relies heavily on **cross-node references** such as `$node["Extract Video Metadata"]`. Renaming nodes after building may break expressions.
- The exported JSON does **not clearly expose binary mappings** for Telegram video sending or YouTube upload. These must be manually checked and configured.
- The fal.ai polling loop currently only exits on `COMPLETED`. You should strongly consider adding explicit handling for `FAILED`, `CANCELLED`, or timeout conditions.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate AI historical videos and upload to YouTube with Telegram approval | Global workflow concept |
| Intended for content creators, YouTube automation builders, educators, and users wanting an automated AI video pipeline with human approval | Workflow purpose |
| Setup reminders included in the workflow notes: configure Telegram credentials, avoid hardcoding chat ID, configure fal.ai header auth, set up Gemini credentials, connect YouTube OAuth2, optionally adjust schedule, then activate | Operational setup |
| Requirements listed in the workflow notes: n8n, fal.ai account/API key, Google Gemini access, YouTube upload permissions, Telegram account | Environment prerequisites |
| Suggested customizations listed in the workflow notes: adjust retry limits, modify fal.ai video parameters, change Gemini prompt style, replace history API, customize Telegram approval flow | Customization guidance |

## Additional Technical Notes
- The workflow contains **one entry point**: `Schedule (Daily at 1 AM)`.
- The workflow contains **no sub-workflow nodes** and does not invoke other workflows.
- Several Telegram nodes use a **hardcoded chat ID**, which is functional but not ideal for portability.
- The polling logic has **no failure-state branch** for fal.ai terminal errors.
- The retry mechanism is conceptually sound, but because approval/rejection data shape may differ between Telegram responses, the loop should be validated in a live test.
- The workflow description embedded in sticky notes is useful operational documentation and should be preserved when migrating the workflow.