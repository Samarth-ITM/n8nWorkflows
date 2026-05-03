Convert Google Sheets scripts into HeyGen AI videos

https://n8nworkflows.xyz/workflows/convert-google-sheets-scripts-into-heygen-ai-videos-14804


# Convert Google Sheets scripts into HeyGen AI videos

# 1. Workflow Overview

This workflow monitors a Google Sheet for updated rows containing a script, sends that script to HeyGen to generate an AI video, polls HeyGen until the video is ready, and then writes the final video URL back into the same sheet.

Its main use case is batch or semi-automated short-form video generation from spreadsheet-managed text content. It is especially suited for teams managing scripts in Google Sheets and wanting automated avatar-video output through HeyGen.

## 1.1 Input Reception and Eligibility Filtering
The workflow starts from a Google Sheets row update trigger. It checks whether the updated row contains a non-empty `Script` field and whether the `Status` is not already `Completed`.

## 1.2 Sequential Processing and Script Sanitization
Eligible rows are processed one at a time. The script text is cleaned to avoid malformed JSON when injected into the HeyGen API request body.

## 1.3 HeyGen Video Creation
The sanitized script is submitted to HeyGen through an HTTP request that defines the avatar, voice, and output dimensions.

## 1.4 Polling for Completion
After the generation request, the workflow waits, checks the video status, and loops until HeyGen reports the video as completed.

## 1.5 Spreadsheet Update and Loop Continuation
Once complete, the workflow updates the original Google Sheet row with `Completed` status, the generated video URL, and a completion timestamp, then returns to the batch node for any remaining items.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Data Processing

### Overview
This block listens for row updates in Google Sheets, filters for rows that actually need processing, and ensures only one item is handled at a time. It prepares the raw spreadsheet data for safe downstream API use.

### Nodes Involved
- Google Sheets Trigger
- Has Script?
- Process One at a Time
- Sanitize Script

### Node Details

#### 1. Google Sheets Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`  
  Trigger node that polls a Google Sheet for row updates.
- **Configuration choices:**
  - Event: `rowUpdate`
  - Polling frequency: every minute
  - Document ID: placeholder value `YOUR_RESOURCE_ID_HERE`
  - Sheet Name: placeholder value `YOUR_RESOURCE_ID_HERE`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to `Has Script?`
- **Version-specific requirements:**
  - Uses typeVersion `1`
  - Requires Google Sheets credentials configured in n8n
- **Edge cases or potential failure types:**
  - Google authentication failure
  - Incorrect spreadsheet or sheet selection
  - Trigger delay due to polling model
  - Row updates unrelated to script generation may still trigger the flow
- **Sub-workflow reference:** None

#### 2. Has Script?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate to determine whether a row should be processed.
- **Configuration choices:**
  - Condition 1: `Script` must not be empty
  - Condition 2: `Status` must not equal `Completed`
- **Key expressions or variables used:**
  - `={{ $json.Script }}`
  - `={{ $json.Status }}`
- **Input and output connections:**
  - Input from `Google Sheets Trigger`
  - True branch goes to `Process One at a Time`
  - False branch unused
- **Version-specific requirements:**
  - Uses typeVersion `1`
- **Edge cases or potential failure types:**
  - If the incoming row schema differs from expected column names, expressions will evaluate incorrectly
  - If `Status` uses values like `complete`, `done`, or lowercase variants, rows may be reprocessed unexpectedly
  - Empty strings with spaces may still pass or fail depending on source formatting
- **Sub-workflow reference:** None

#### 3. Process One at a Time
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Batch-control node used here to serialize processing so only one row runs through the HeyGen cycle at a time.
- **Configuration choices:**
  - No custom options configured
  - Default batching behavior
- **Key expressions or variables used:** Later referenced by other nodes:
  - `$('Process One at a Time').item.json.Row_ID`
- **Input and output connections:**
  - Input from `Has Script?`
  - Batch output index 1 goes to `Sanitize Script`
  - Loop continuation input receives data from `Update Sheet with Video1`
- **Version-specific requirements:**
  - Uses typeVersion `3`
- **Edge cases or potential failure types:**
  - If the trigger usually emits only one row per event, batching adds control but limited throughput benefit
  - Incorrect loop wiring can stall or duplicate processing
- **Sub-workflow reference:** None

#### 4. Sanitize Script
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that cleans script text for safe JSON embedding.
- **Configuration choices:**
  - Reads `item.Script`
  - Produces `CleanScript`
  - Replaces:
    - Windows/Unix/Mac line breaks with spaces
    - tabs with spaces
    - backslashes with escaped backslashes
    - double quotes with escaped quotes
    - repeated whitespace with a single space
    - trims leading/trailing spaces
- **Key expressions or variables used:**
  - `const item = $input.item.json;`
  - `const rawScript = item.Script || '';`
  - Output field: `CleanScript`
- **Input and output connections:**
  - Input from `Process One at a Time`
  - Output to `Generate Video`
- **Version-specific requirements:**
  - Uses code node typeVersion `2`
  - Requires JavaScript execution enabled as standard in n8n
- **Edge cases or potential failure types:**
  - If `Script` contains very long content, API body size may become an issue
  - Escaping is useful here because the body is assembled as expression-based JSON text
  - If later switched to structured JSON mode without string interpolation, some sanitization steps may become redundant
- **Sub-workflow reference:** None

---

## Block 2 — API Communication

### Overview
This block sends the cleaned script to HeyGen for video generation, then repeatedly checks the generation status until completion. It is the core integration layer between n8n and the HeyGen API.

### Nodes Involved
- Generate Video
- If
- Wait
- Video Status
- Wait Before Retry1

### Node Details

#### 5. Generate Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the HeyGen video-generation endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: placeholder `YOUR_API_ENDPOINT_HERE`
  - Body format: JSON
  - Headers manually supplied:
    - `X-Api-Key`
    - `Content-Type: application/json`
  - Request body includes:
    - `video_inputs`
    - avatar character type and hardcoded `avatar_id`
    - text voice type and hardcoded `voice_id`
    - `input_text` from `{{ $json.CleanScript }}`
    - dimensions `720 x 1280`
- **Key expressions or variables used:**
  - `{{ $json.CleanScript }}`
- **Input and output connections:**
  - Input from `Sanitize Script`
  - Output to `If`
- **Version-specific requirements:**
  - Uses HTTP Request typeVersion `4.3`
- **Edge cases or potential failure types:**
  - Invalid or expired API key
  - Wrong HeyGen endpoint
  - Invalid avatar or voice IDs
  - Request body rejected due to unsupported format or model requirements
  - Hardcoded secret in node configuration is a security risk
  - API quota/rate limit issues
- **Sub-workflow reference:** None

#### 6. If
- **Type and technical role:** `n8n-nodes-base.if`  
  Intended as a post-request decision gate before entering the wait-and-poll cycle.
- **Configuration choices:**
  - The current condition appears misconfigured:
    - Left value is `=`
    - Right value is empty
    - Operator is string equals
- **Key expressions or variables used:** None meaningfully usable in current state.
- **Input and output connections:**
  - Input from `Generate Video`
  - True branch goes to `Wait`
  - False branch unused
- **Version-specific requirements:**
  - Uses typeVersion `2.3`
- **Edge cases or potential failure types:**
  - As configured, this node is likely non-functional or always false/always unpredictable depending on n8n parsing behavior
  - The workflow may fail to continue after video creation because this condition does not actually inspect the API response
- **Sub-workflow reference:** None

**Important interpretation:**  
This node should likely check whether the HeyGen creation request succeeded and whether `data.video_id` exists. A more robust condition would be based on the existence of `{{$json.data.video_id}}`.

#### 7. Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays execution before the first status check.
- **Configuration choices:**
  - Wait amount: `60`
  - No explicit unit shown in the JSON; in n8n this commonly means a duration value configured in the node UI and should be confirmed when rebuilding
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `If`
  - Output to `Video Status`
- **Version-specific requirements:**
  - Uses typeVersion `1.1`
- **Edge cases or potential failure types:**
  - If the wait is too short, polling frequency may be unnecessarily aggressive
  - If too long, turnaround time increases
  - Wait node behavior depends on n8n execution mode and persistence setup
- **Sub-workflow reference:** None

#### 8. Video Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries HeyGen for the current status of a previously requested video.
- **Configuration choices:**
  - URL: placeholder `YOUR_API_ENDPOINT_HERE`
  - Sends query parameter `video_id`
  - Uses generic credential type: HTTP Header Auth
  - Adds header `accept: application/json`
- **Key expressions or variables used:**
  - `={{ $('Generate Video').item.json.data.video_id }}`
- **Input and output connections:**
  - Inputs from `Wait` and `Wait Before Retry1`
  - Output to `Is Complete?1`
- **Version-specific requirements:**
  - Uses HTTP Request typeVersion `4.3`
  - Requires a generic HTTP header auth credential configured in n8n
- **Edge cases or potential failure types:**
  - If `Generate Video` did not return `data.video_id`, this expression will fail
  - If execution is resumed after a long wait and item linking changes, cross-node item resolution can fail in some designs
  - Authentication mismatch between the create endpoint and status endpoint can cause confusing partial success
- **Sub-workflow reference:** None

#### 9. Wait Before Retry1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delay node for repeated polling when the video is not yet complete.
- **Configuration choices:**
  - Wait amount: `60`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from false branch of `Is Complete?1`
  - Output back to `Video Status`
- **Version-specific requirements:**
  - Uses typeVersion `1.1`
- **Edge cases or potential failure types:**
  - No maximum retry count is implemented
  - If HeyGen leaves a job in a failed or stuck state, this loop can continue indefinitely
- **Sub-workflow reference:** None

---

## Block 3 — Sheet Update

### Overview
This block determines whether the video is ready and, when it is, writes the completion data back to the originating row in Google Sheets. It closes the processing loop for each row.

### Nodes Involved
- Is Complete?1
- Update Sheet with Video1

### Node Details

#### 10. Is Complete?1
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks the status response from HeyGen.
- **Configuration choices:**
  - Compares `data.status` to the literal value `completed`
- **Key expressions or variables used:**
  - `={{ $json.data.status }}`
- **Input and output connections:**
  - Input from `Video Status`
  - True branch to `Update Sheet with Video1`
  - False branch to `Wait Before Retry1`
- **Version-specific requirements:**
  - Uses typeVersion `1`
- **Edge cases or potential failure types:**
  - Case-sensitive comparison may fail if API returns `Completed` or another variant
  - Failed/error statuses are not handled separately
  - Any non-completed terminal state will cause indefinite retrying
- **Sub-workflow reference:** None

#### 11. Update Sheet with Video1
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the source Google Sheet row with completion metadata.
- **Configuration choices:**
  - Operation: `update`
  - Match row using `Row_ID`
  - Writes:
    - `Row_ID` from the batch node item
    - `Status` = `Completed`
    - `Video_URL` from `data.video_url`
    - `Completed_At` from current timestamp
  - `row_number` included in schema but set to `0` in mapped value
- **Key expressions or variables used:**
  - `={{ $('Process One at a Time').item.json.Row_ID }}`
  - `={{ $json.data.video_url }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input from true branch of `Is Complete?1`
  - Output back to `Process One at a Time`
- **Version-specific requirements:**
  - Uses Google Sheets node typeVersion `4.5`
  - Requires Google Sheets credentials
- **Edge cases or potential failure types:**
  - If `Row_ID` is missing or duplicated, the wrong row may be updated or no row may be matched
  - If the sheet schema changes, mapping can break
  - `row_number: 0` is present but matching is based on `Row_ID`; this may be harmless but is not necessary
  - If `data.video_url` is absent despite completed status, the row may be marked completed without a usable URL
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | Google Sheets Trigger | Polls Google Sheets for updated rows |  | Has Script? | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 1. Trigger & Data Processing<br>Watches Google Sheets for pending scripts and cleans the text. |
| Has Script? | If | Filters rows with a script that are not completed | Google Sheets Trigger | Process One at a Time | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 1. Trigger & Data Processing<br>Watches Google Sheets for pending scripts and cleans the text. |
| Process One at a Time | Split In Batches | Serializes row processing and supports looping | Has Script?, Update Sheet with Video1 | Sanitize Script | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 1. Trigger & Data Processing<br>Watches Google Sheets for pending scripts and cleans the text. |
| Sanitize Script | Code | Cleans and escapes script text for JSON-safe API submission | Process One at a Time | Generate Video | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 1. Trigger & Data Processing<br>Watches Google Sheets for pending scripts and cleans the text. |
| Generate Video | HTTP Request | Sends video generation request to HeyGen | Sanitize Script | If | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 2. API Communication<br>Handles HeyGen requests and polls for video generation status. |
| If | If | Intended validation gate after video creation request | Generate Video | Wait | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 2. API Communication<br>Handles HeyGen requests and polls for video generation status. |
| Wait | Wait | Delays before first HeyGen status check | If | Video Status | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 2. API Communication<br>Handles HeyGen requests and polls for video generation status. |
| Video Status | HTTP Request | Polls HeyGen for video generation status | Wait, Wait Before Retry1 | Is Complete?1 | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 2. API Communication<br>Handles HeyGen requests and polls for video generation status. |
| Is Complete?1 | If | Checks whether HeyGen has finished rendering the video | Video Status | Update Sheet with Video1, Wait Before Retry1 | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 3. Sheet Update<br>Writes the generated video URL back to the spreadsheet. |
| Update Sheet with Video1 | Google Sheets | Marks row completed and writes video URL and timestamp | Is Complete?1 | Process One at a Time | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 3. Sheet Update<br>Writes the generated video URL back to the spreadsheet. |
| Wait Before Retry1 | Wait | Waits before polling HeyGen again when not complete | Is Complete?1 | Video Status | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration.<br>## 3. Sheet Update<br>Writes the generated video URL back to the spreadsheet. |
| Main Sticky | Sticky Note | Documentation note for the workflow canvas |  |  | ## Automate HeyGen Video Generation from Google Sheets<br>Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen.<br><br>### How it works<br>1. Monitor Google Sheets for new row updates.<br>2. Sanitize script text to prevent JSON errors.<br>3. Send text to HeyGen API for processing.<br>4. Poll for video completion status.<br>5. Update Google Sheets with the final video URL.<br><br>### Setup<br>1. Connect your Google Sheets account.<br>2. Provide your HeyGen API Key in the HTTP Request node.<br>3. Configure Document ID and Sheet Name in Google nodes.<br>4. Adjust the avatar and voice IDs as needed.<br><br>### Customization<br>Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration. |
| Section 1 | Sticky Note | Visual section label for input and preprocessing |  |  | ## 1. Trigger & Data Processing<br>Watches Google Sheets for pending scripts and cleans the text. |
| Section 2 | Sticky Note | Visual section label for API communication |  |  | ## 2. API Communication<br>Handles HeyGen requests and polls for video generation status. |
| Section 3 | Sticky Note | Visual section label for sheet update |  |  | ## 3. Sheet Update<br>Writes the generated video URL back to the spreadsheet. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites
1. Have access to:
   - a Google account with the target spreadsheet
   - a HeyGen API key
   - the correct HeyGen API endpoints for:
     - video creation
     - video status lookup
2. Prepare a Google Sheet with at least these columns:
   - `Row_ID`
   - `Script`
   - `Status`
   - `Video_URL`
   - `Completed_At`
3. Ensure each row has a unique `Row_ID`.

## Build Steps

1. **Create a new workflow**  
   Name it: **Convert Google Sheets scripts into HeyGen AI videos**

2. **Add a Google Sheets Trigger node**
   - Node type: **Google Sheets Trigger**
   - Event: **Row Update**
   - Polling: **Every Minute**
   - Select your Google Sheets credential
   - Select the target spreadsheet in **Document ID**
   - Select the target tab in **Sheet Name**

3. **Add an If node named `Has Script?`**
   - Connect `Google Sheets Trigger` → `Has Script?`
   - Add two string conditions:
     - `Script` **is not empty**
     - `Status` **not equal** `Completed`
   - Use expressions:
     - `{{$json.Script}}`
     - `{{$json.Status}}`

4. **Add a Split In Batches node named `Process One at a Time`**
   - Connect the true output of `Has Script?` → `Process One at a Time`
   - Keep default settings unless you want to explicitly set batch size to 1
   - Use this node to serialize processing

5. **Add a Code node named `Sanitize Script`**
   - Connect `Process One at a Time` batch output to `Sanitize Script`
   - Paste the following logic conceptually:
     - read `Script`
     - replace line breaks and tabs with spaces
     - escape backslashes
     - escape double quotes
     - collapse repeated whitespace
     - store result in `CleanScript`
   - The node should return the original JSON plus `CleanScript`

6. **Add an HTTP Request node named `Generate Video`**
   - Connect `Sanitize Script` → `Generate Video`
   - Method: **POST**
   - URL: set to HeyGen’s video-generation endpoint
   - Enable **Send Headers**
   - Enable **Send Body**
   - Body type: **JSON**
   - Add headers:
     - `X-Api-Key`: your HeyGen API key
     - `Content-Type`: `application/json`
   - Add a JSON body equivalent to:
     - `video_inputs[0].character.type = avatar`
     - `video_inputs[0].character.avatar_id = your avatar ID`
     - `video_inputs[0].voice.type = text`
     - `video_inputs[0].voice.input_text = {{$json.CleanScript}}`
     - `video_inputs[0].voice.voice_id = your voice ID`
     - `dimension.width = 720`
     - `dimension.height = 1280`

7. **Add a validation If node after creation request**
   - Name it `If` to match the original, though a better name would be `Video Request OK?`
   - Connect `Generate Video` → `If`
   - The original workflow contains a broken condition; when rebuilding, configure it properly:
     - Check that `{{$json.data.video_id}}` is not empty
   - Connect the true output to the next wait step
   - Optionally connect the false output to an error-handling path or logging node

8. **Add a Wait node named `Wait`**
   - Connect true output of `If` → `Wait`
   - Set wait duration to **60 seconds**

9. **Add an HTTP Request node named `Video Status`**
   - Connect `Wait` → `Video Status`
   - Method: typically **GET** unless HeyGen requires another method
   - URL: set to HeyGen’s video-status endpoint
   - Authentication:
     - either use **HTTP Header Auth** credential
     - or manually send the same header required by HeyGen
   - Add query parameter:
     - `video_id = {{$('Generate Video').item.json.data.video_id}}`
   - Add header:
     - `accept = application/json`

10. **Add an If node named `Is Complete?1`**
    - Connect `Video Status` → `Is Complete?1`
    - Condition:
      - `{{$json.data.status}}` equals `completed`

11. **Add a Google Sheets node named `Update Sheet with Video1`**
    - Connect true output of `Is Complete?1` → `Update Sheet with Video1`
    - Select the same Google Sheets credential
    - Operation: **Update**
    - Select the same spreadsheet and sheet
    - Match using column:
      - `Row_ID`
    - Map fields:
      - `Row_ID = {{$('Process One at a Time').item.json.Row_ID}}`
      - `Status = Completed`
      - `Video_URL = {{$json.data.video_url}}`
      - `Completed_At = {{$now.toISO()}}`
    - If schema mapping is required, ensure these columns exist in the sheet

12. **Loop back to the batch node**
    - Connect `Update Sheet with Video1` → `Process One at a Time`
    - This allows the next item to continue if multiple rows are present

13. **Add a second Wait node named `Wait Before Retry1`**
    - Connect false output of `Is Complete?1` → `Wait Before Retry1`
    - Set wait duration to **60 seconds**

14. **Create the polling loop**
    - Connect `Wait Before Retry1` → `Video Status`
    - This forms the status polling cycle

15. **Optionally add sticky notes**
    - Add a top-level note describing:
      - workflow purpose
      - setup instructions
      - customization advice
    - Add section labels:
      - `1. Trigger & Data Processing`
      - `2. API Communication`
      - `3. Sheet Update`

---

## Credential Configuration

### Google Sheets
- Create a Google Sheets credential in n8n
- Authorize the account that owns or can edit the spreadsheet
- Use the same credential for:
  - `Google Sheets Trigger`
  - `Update Sheet with Video1`

### HeyGen API
The workflow uses two patterns:
- manual `X-Api-Key` header in `Generate Video`
- generic HTTP Header Auth in `Video Status`

For consistency, it is better to standardize on one approach:
- either use manual headers in both nodes
- or use an HTTP Header Auth credential in both nodes

Recommended header:
- `X-Api-Key: <your_api_key>`

---

## Recommended Improvements During Rebuild

1. **Replace hardcoded API key**
   - Do not store the key directly in the node
   - Use credentials or a config node

2. **Fix the broken `If` node after `Generate Video`**
   - Check for `data.video_id`
   - Or check HTTP status / success flag if returned by HeyGen

3. **Handle failure states explicitly**
   - If HeyGen returns statuses like `failed`, `error`, or `canceled`, stop retrying
   - Update the sheet with an error status instead

4. **Add retry limits**
   - Store and increment a retry counter
   - Stop after a maximum number of polls

5. **Use a Set node for configuration**
   - Store reusable values:
     - avatar ID
     - voice ID
     - API base URL
     - polling interval
     - sheet names

6. **Normalize status values**
   - Consider matching status case-insensitively

7. **Avoid over-escaping if using native JSON body mode**
   - If you build the request body with structured fields instead of raw JSON string interpolation, the sanitization can be simplified

---

## Sub-workflow Setup
This workflow does **not** contain any sub-workflow nodes and does **not** invoke another workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automate HeyGen Video Generation from Google Sheets — Effortlessly convert text scripts in Google Sheets into AI-generated videos using HeyGen. | Workflow purpose |
| How it works: 1. Monitor Google Sheets for new row updates. 2. Sanitize script text to prevent JSON errors. 3. Send text to HeyGen API for processing. 4. Poll for video completion status. 5. Update Google Sheets with the final video URL. | Workflow logic summary |
| Setup: 1. Connect your Google Sheets account. 2. Provide your HeyGen API Key in the HTTP Request node. 3. Configure Document ID and Sheet Name in Google nodes. 4. Adjust the avatar and voice IDs as needed. | Configuration guidance |
| Customization: Consolidate user-specific values (API Keys, IDs) in a Set node at the workflow start for easy configuration. | Recommended design improvement |
| 1. Trigger & Data Processing — Watches Google Sheets for pending scripts and cleans the text. | Canvas section note |
| 2. API Communication — Handles HeyGen requests and polls for video generation status. | Canvas section note |
| 3. Sheet Update — Writes the generated video URL back to the spreadsheet. | Canvas section note |

## Additional Technical Note
The provided workflow contains a likely configuration defect in the node named `If` located after `Generate Video`. Reproducing the workflow exactly will preserve that defect; rebuilding it for production use should replace that condition with a real success check based on the HeyGen response, ideally verifying `data.video_id` before entering the polling loop.