Analyze Zoom phone call recordings with Gemini and log results to Google Sheets

https://n8nworkflows.xyz/workflows/analyze-zoom-phone-call-recordings-with-gemini-and-log-results-to-google-sheets-13871


# Analyze Zoom phone call recordings with Gemini and log results to Google Sheets

# 1. Workflow Overview

This workflow automatically retrieves recent **Zoom Phone call recordings**, processes each recording one by one, **backs up the audio to Google Drive**, sends the audio to **Google Gemini** for structured call analysis, and finally **logs the results to Google Sheets**.

Its primary use cases are:

- Sales call review and quality monitoring
- Telemarketing call scoring
- Lightweight conversation outcome tracking
- Building a searchable log of calls plus AI-generated coaching feedback

The workflow is organized into the following logical blocks:

## 1.1 Trigger and Retrieval
A scheduled trigger runs every hour and calls the Zoom Phone Recordings API to fetch recent recordings.

## 1.2 Item Expansion and Iteration
The returned recordings array is split into individual items, then processed sequentially in a loop.

## 1.3 File Naming and Audio Download
For each recording, a filename is generated from caller metadata and timestamp, and the audio file is downloaded from Zoom.

## 1.4 Parallel Backup and AI Analysis
The downloaded audio is sent in parallel to:
- Google Drive for archival storage
- Gemini File Upload API, then Gemini Generate Content API for structured analysis

## 1.5 Result Consolidation and Spreadsheet Logging
The Google Drive link and Gemini analysis result are merged, normalized into spreadsheet-ready fields, and appended to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Retrieval

### Overview
This block starts the workflow on a schedule and retrieves a batch of phone recordings from Zoom. It is the entry point for the entire automation.

### Nodes Involved
- Run Every Hour
- Get Zoom Phone Recordings

### Node Details

#### Run Every Hour
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring interval.
- **Configuration choices:**  
  Configured to run every hour using an interval rule on the `hours` field.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input; this is a trigger node.
  - Output → `Get Zoom Phone Recordings`
- **Version-specific requirements:**  
  Uses node version `1.3`.
- **Edge cases or potential failure types:**  
  - Workflow will not run unless activated.
  - Timezone behavior depends on workflow/server settings.
  - If frequent polling is undesirable, the schedule should be adjusted.

#### Get Zoom Phone Recordings
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Zoom Phone API to fetch recordings.
- **Configuration choices:**  
  - Method defaults to GET
  - URL: `https://api.zoom.us/v2/phone/recordings`
  - Authentication uses predefined Zoom OAuth2 credentials
  - Query parameter `page_size=20`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input ← `Run Every Hour`
  - Output → `Process Each Recording`
- **Version-specific requirements:**  
  Uses node version `4.2`.
- **Edge cases or potential failure types:**  
  - Zoom OAuth token missing, expired, or misconfigured
  - Missing required Zoom scopes:
    - `phone:read`
    - `recording:read`
  - API pagination is not fully handled beyond `page_size=20`
  - Empty API response or missing `recordings` field
  - Zoom rate limiting or temporary API outage
- **Sub-workflow reference:**  
  None.

---

## 2.2 Item Expansion and Iteration

### Overview
This block converts the array of recordings returned by Zoom into one item per recording and processes them sequentially. It ensures downstream nodes handle one call recording at a time.

### Nodes Involved
- Process Each Recording
- Loop Over Items

### Node Details

#### Process Each Recording
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `recordings` array from the Zoom response into separate n8n items.
- **Configuration choices:**  
  - `fieldToSplitOut`: `recordings`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input ← `Get Zoom Phone Recordings`
  - Output → `Loop Over Items`
- **Version-specific requirements:**  
  Uses node version `1`.
- **Edge cases or potential failure types:**  
  - If Zoom returns no `recordings` field, this node may produce no items or fail depending on the actual payload shape
  - If `recordings` is empty, nothing else will run
- **Sub-workflow reference:**  
  None.

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the split recordings one by one.
- **Configuration choices:**  
  Default options are used; no explicit batch size is defined in the exported parameters.
- **Key expressions or variables used:**  
  Several downstream nodes reference this node’s current item using expressions like:
  - `$('Loop Over Items').item.json.download_url`
  - `$('Loop Over Items').item.json.caller_name`
- **Input and output connections:**  
  - Input ← `Process Each Recording`
  - Output 1 (continue branch) is not used directly
  - Output 2 → `Build File Name`
  - Input from `Log to Google Sheets` closes the loop for next item
- **Version-specific requirements:**  
  Uses node version `3`.
- **Edge cases or potential failure types:**  
  - If any item fails downstream, loop progression may stop unless error handling is added
  - Large batches can increase execution time
  - Misunderstanding output branch behavior can lead to broken loops when rebuilding
- **Sub-workflow reference:**  
  None.

---

## 2.3 File Naming and Audio Download

### Overview
This block creates a deterministic filename for each recording and downloads the associated audio file from Zoom. The downloaded binary is later reused for both storage and AI analysis.

### Nodes Involved
- Build File Name
- Download Recording

### Node Details

#### Build File Name
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a new field containing the generated filename.
- **Configuration choices:**  
  Adds one string field:
  - `FileName = {{ $json.caller_name }}_{{ $json.date_time.replace(/[-T:Z]/g, '') }}`
- **Key expressions or variables used:**  
  - `$json.caller_name`
  - `$json.date_time.replace(/[-T:Z]/g, '')`
- **Input and output connections:**  
  - Input ← `Loop Over Items`
  - Output → `Download Recording`
- **Version-specific requirements:**  
  Uses node version `3.4`.
- **Edge cases or potential failure types:**  
  - If `caller_name` is null or undefined, filename may be incomplete
  - If `date_time` is missing or not a string, the `.replace(...)` expression may fail
  - Caller names may contain characters unsuitable for filenames depending on downstream service constraints
- **Sub-workflow reference:**  
  None.

#### Download Recording
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the actual audio file from Zoom as binary data.
- **Configuration choices:**  
  - URL is taken dynamically from the current loop item:
    `={{ $('Loop Over Items').item.json.download_url }}`
  - Authentication uses Zoom OAuth2
  - Response format is set to `file`
- **Key expressions or variables used:**  
  - `$('Loop Over Items').item.json.download_url`
- **Input and output connections:**  
  - Input ← `Build File Name`
  - Output → `Upload to Google Drive`
  - Output → `Upload Audio to Gemini`
- **Version-specific requirements:**  
  Uses node version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing `download_url`
  - Zoom authentication may be required for download endpoint access
  - Large files may exceed time or memory limits
  - Non-audio files or unexpected content type from Zoom
  - Binary property naming must remain compatible with downstream nodes
- **Sub-workflow reference:**  
  None.

---

## 2.4 Parallel Backup and AI Analysis

### Overview
This block branches the downloaded audio into two parallel paths. One path uploads the recording to Google Drive for storage, while the other path uploads the file to Gemini and requests a structured JSON analysis.

### Nodes Involved
- Upload to Google Drive
- Upload Audio to Gemini
- Analyze Call with Gemini

### Node Details

#### Upload to Google Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the downloaded recording file into Google Drive.
- **Configuration choices:**  
  - File name comes from:
    `={{ $('Build File Name').item.json.FileName }}`
  - Drive is set to `My Drive`
  - Folder ID is required but currently blank in the exported workflow
- **Key expressions or variables used:**  
  - `$('Build File Name').item.json.FileName`
- **Input and output connections:**  
  - Input ← `Download Recording`
  - Output → `Merge Drive Link + Analysis`
- **Version-specific requirements:**  
  Uses node version `3`.
- **Edge cases or potential failure types:**  
  - Folder ID must be set before use
  - Google Drive OAuth2 credential must be configured
  - Duplicate file names may occur
  - Large file upload may timeout
  - If the node does not receive the expected binary property, upload will fail
- **Sub-workflow reference:**  
  None.

#### Upload Audio to Gemini
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Uploads the binary audio file to the Gemini Files API.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`
  - Authentication uses Google PaLM / Gemini API credentials
  - Content type: binary data
  - Binary input field: `data`
  - Headers:
    - `X-Goog-Upload-Command: upload, finalize`
    - `X-Goog-Upload-Protocol: raw`
- **Key expressions or variables used:**  
  None beyond the fixed binary field selection.
- **Input and output connections:**  
  - Input ← `Download Recording`
  - Output → `Analyze Call with Gemini`
- **Version-specific requirements:**  
  Uses node version `4.4`.
- **Edge cases or potential failure types:**  
  - Requires Gemini-compatible API key/credential
  - If the downloaded binary property is not named `data`, upload will fail
  - Unsupported file type or file too large for Gemini upload
  - Google API quota/rate limit issues
  - Upload API behavior may evolve across Gemini API versions
- **Sub-workflow reference:**  
  None.

#### Analyze Call with Gemini
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a multimodal prompt referencing the uploaded file and requests structured JSON output.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
  - Authentication uses Google PaLM / Gemini API credential
  - Body is JSON
  - Includes:
    - `fileData.mimeType` from upload response
    - `fileData.fileUri` from upload response
    - prompt instructing Gemini to analyze a sales/telemarketing call
  - Enforces structured output through:
    - `responseMimeType: application/json`
    - `responseSchema` with required fields:
      - `step`
      - `advice`
- **Key expressions or variables used:**  
  - `{{ $json.file.mimeType }}`
  - `{{ $json.file.uri }}`
- **Input and output connections:**  
  - Input ← `Upload Audio to Gemini`
  - Output → `Merge Drive Link + Analysis`
- **Version-specific requirements:**  
  Uses node version `4.4`.
- **Edge cases or potential failure types:**  
  - Gemini may still return malformed or partial data despite schema guidance
  - If `file.mimeType` or `file.uri` is absent, the request fails
  - Audio quality, language, noise, or duration may affect analysis quality
  - Prompt assumes the recording is a sales/telemarketing call; other call types may yield poor classification
  - API model availability may change over time
- **Sub-workflow reference:**  
  None.

---

## 2.5 Result Consolidation and Spreadsheet Logging

### Overview
This block merges the Google Drive upload result with the Gemini analysis result, extracts normalized fields, and appends them to a Google Sheets tracking log.

### Nodes Involved
- Merge Drive Link + Analysis
- Format Results
- Log to Google Sheets

### Node Details

#### Merge Drive Link + Analysis
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the outputs of the Google Drive branch and Gemini branch into one item.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine strategy: `combineAll`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input 1 ← `Upload to Google Drive`
  - Input 2 ← `Analyze Call with Gemini`
  - Output → `Format Results`
- **Version-specific requirements:**  
  Uses node version `3.2`.
- **Edge cases or potential failure types:**  
  - If one branch fails or returns no item, merge may stall or output incomplete data
  - Assumes one corresponding item from each branch per recording
- **Sub-workflow reference:**  
  None.

#### Format Results
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps merged data into clean spreadsheet fields.
- **Configuration choices:**  
  Creates the following fields:
  - `Caller = {{ $('Loop Over Items').item.json.caller_name }}`
  - `Date = {{ $('Loop Over Items').item.json.date_time }}`
  - `Number = {{ $('Loop Over Items').item.json.callee_number }}`
  - `Direction = {{ $('Loop Over Items').item.json.direction }}`
  - `Stage = {{ JSON.parse($json.candidates[0].content.parts[0].text).step }}`
  - `Advice = {{ JSON.parse($json.candidates[0].content.parts[0].text).advice }}`
  - `DriveURL = {{ $json.webViewLink }}`
- **Key expressions or variables used:**  
  - `$('Loop Over Items').item.json.caller_name`
  - `$('Loop Over Items').item.json.date_time`
  - `$('Loop Over Items').item.json.callee_number`
  - `$('Loop Over Items').item.json.direction`
  - `JSON.parse($json.candidates[0].content.parts[0].text).step`
  - `JSON.parse($json.candidates[0].content.parts[0].text).advice`
  - `$json.webViewLink`
- **Input and output connections:**  
  - Input ← `Merge Drive Link + Analysis`
  - Output → `Log to Google Sheets`
- **Version-specific requirements:**  
  Uses node version `3.4`.
- **Edge cases or potential failure types:**  
  - `JSON.parse(...)` will fail if Gemini returns non-JSON or invalid JSON
  - `candidates[0].content.parts[0].text` may be absent
  - `webViewLink` may not be available depending on Drive node response behavior
  - Missing caller metadata fields from Zoom will create blank spreadsheet values
- **Sub-workflow reference:**  
  None.

#### Log to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row per processed call to a Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet name: `Sheet1`
  - Document ID is required but currently blank in the exported workflow
  - Column mapping explicitly defines:
    - Caller
    - Date
    - Number
    - Direction
    - Stage
    - Advice
    - DriveURL
- **Key expressions or variables used:**  
  - `={{ $json.Caller }}`
  - `={{ $json.Date }}`
  - `={{ $json.Number }}`
  - `={{ $json.Direction }}`
  - `={{ $json.Stage }}`
  - `={{ $json.Advice }}`
  - `={{ $json.DriveURL }}`
- **Input and output connections:**  
  - Input ← `Format Results`
  - Output → `Loop Over Items` to continue batch iteration
- **Version-specific requirements:**  
  Uses node version `4.5`.
- **Edge cases or potential failure types:**  
  - Google Sheets OAuth2 credential must be configured
  - Document ID must be set
  - Sheet name must exist
  - Header names should match expected columns
  - Permission issues on the target spreadsheet can prevent append operations
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Every Hour | Schedule Trigger | Starts the workflow every hour |  | Get Zoom Phone Recordings | ### Analyze Zoom phone recordings with Gemini AI<br>This workflow grabs your Zoom Phone call recordings, sends them to Gemini for analysis, backs them up to Google Drive, and logs everything in a spreadsheet.<br><br>### How it works<br>1. **Schedule Trigger** kicks off every few hours and pulls recent recordings from Zoom Phone API<br>2. Each recording is downloaded, then sent to both **Google Drive** (for backup) and **Gemini AI** (for analysis)<br>3. Gemini evaluates the call — how far the conversation got, and what could be improved<br>4. Results are merged with the Drive link and appended to a **Google Sheets** log<br><br>### Setup steps<br>1. Create a Zoom Server-to-Server OAuth app with `phone:read` and `recording:read` scopes<br>2. Add your Google Gemini API key in n8n credentials<br>3. Connect Google Drive and Google Sheets OAuth2<br>4. Set the **Google Drive folder ID** in the "Upload to Google Drive" node<br>5. Set the **Google Sheets document ID** in the "Log to Google Sheets" node — make sure the sheet has columns: Caller, Date, Number, Direction, Stage, Advice, Drive URL<br><br>### Customization<br>- Change the schedule interval in the trigger node<br>- Edit the Gemini prompt to match your call scoring criteria<br>- Add more columns to the spreadsheet for extra tracking |
| Get Zoom Phone Recordings | HTTP Request | Fetches recent Zoom Phone recordings | Run Every Hour | Process Each Recording | ## Fetch Zoom recordings<br>Pulls recent phone recordings from the Zoom API and splits them into individual items for processing. |
| Process Each Recording | Split Out | Splits the `recordings` array into one item per recording | Get Zoom Phone Recordings | Loop Over Items | ## Fetch Zoom recordings<br>Pulls recent phone recordings from the Zoom API and splits them into individual items for processing. |
| Loop Over Items | Split In Batches | Iterates through recordings sequentially | Process Each Recording, Log to Google Sheets | Build File Name | ## Prepare & download<br>Builds a clean filename from the caller name + timestamp, then downloads the actual audio file. |
| Build File Name | Set | Generates a file name from caller and timestamp | Loop Over Items | Download Recording | ## Prepare & download<br>Builds a clean filename from the caller name + timestamp, then downloads the actual audio file. |
| Download Recording | HTTP Request | Downloads the recording file from Zoom | Build File Name | Upload to Google Drive, Upload Audio to Gemini | ## Prepare & download<br>Builds a clean filename from the caller name + timestamp, then downloads the actual audio file. |
| Upload to Google Drive | Google Drive | Stores the recording in Google Drive | Download Recording | Merge Drive Link + Analysis | ## Analyze with Gemini AI<br>The audio goes to both Google Drive (backup) and Gemini (analysis) in parallel. Gemini returns a structured JSON with the call stage and improvement advice. |
| Upload Audio to Gemini | HTTP Request | Uploads binary audio to Gemini Files API | Download Recording | Analyze Call with Gemini | ## Analyze with Gemini AI<br>The audio goes to both Google Drive (backup) and Gemini (analysis) in parallel. Gemini returns a structured JSON with the call stage and improvement advice. |
| Analyze Call with Gemini | HTTP Request | Requests structured call analysis from Gemini | Upload Audio to Gemini | Merge Drive Link + Analysis | ## Analyze with Gemini AI<br>The audio goes to both Google Drive (backup) and Gemini (analysis) in parallel. Gemini returns a structured JSON with the call stage and improvement advice. |
| Merge Drive Link + Analysis | Merge | Combines Drive metadata and Gemini result | Upload to Google Drive, Analyze Call with Gemini | Format Results | ## Analyze with Gemini AI<br>The audio goes to both Google Drive (backup) and Gemini (analysis) in parallel. Gemini returns a structured JSON with the call stage and improvement advice. |
| Format Results | Set | Extracts spreadsheet-ready fields | Merge Drive Link + Analysis | Log to Google Sheets | ## Log results<br>Combines the Gemini analysis with caller metadata and the Drive link, then appends a row to your tracking spreadsheet. |
| Log to Google Sheets | Google Sheets | Appends result rows to the tracking sheet | Format Results | Loop Over Items | ## Log results<br>Combines the Gemini analysis with caller metadata and the Drive link, then appends a row to your tracking spreadsheet. |
| Sticky Note | Sticky Note | Documentation / visual annotation |  |  |  |
| Sticky Note1 | Sticky Note | Documentation / visual annotation |  |  |  |
| Sticky Note2 | Sticky Note | Documentation / visual annotation |  |  |  |
| Sticky Note3 | Sticky Note | Documentation / visual annotation |  |  |  |
| Sticky Note4 | Sticky Note | Documentation / visual annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name it: `Run Every Hour`
   - Configure interval rule to run every hour.

3. **Add an HTTP Request node for Zoom**
   - Node type: `HTTP Request`
   - Name it: `Get Zoom Phone Recordings`
   - Method: `GET`
   - URL: `https://api.zoom.us/v2/phone/recordings`
   - Authentication: predefined credential
   - Credential type: `Zoom OAuth2 API`
   - Add query parameter:
     - `page_size` = `20`
   - Connect `Run Every Hour` → `Get Zoom Phone Recordings`

4. **Configure Zoom credentials**
   - Create or select Zoom OAuth2 credentials in n8n
   - Use a Zoom Server-to-Server OAuth app
   - Ensure scopes include:
     - `phone:read`
     - `recording:read`

5. **Add a Split Out node**
   - Node type: `Split Out`
   - Name it: `Process Each Recording`
   - Field to split out: `recordings`
   - Connect `Get Zoom Phone Recordings` → `Process Each Recording`

6. **Add a Split In Batches node**
   - Node type: `Loop Over Items`
   - Actual node type: `Split In Batches`
   - Keep default settings
   - Connect `Process Each Recording` → `Loop Over Items`

7. **Add a Set node for filename creation**
   - Node type: `Set`
   - Name it: `Build File Name`
   - Add one string field:
     - `FileName`
     - Value:
       `={{ $json.caller_name }}_{{ $json.date_time.replace(/[-T:Z]/g, '') }}`
   - Connect the processing output of `Loop Over Items` → `Build File Name`

8. **Add an HTTP Request node to download audio**
   - Node type: `HTTP Request`
   - Name it: `Download Recording`
   - Method: `GET`
   - URL:
     `={{ $('Loop Over Items').item.json.download_url }}`
   - Authentication: predefined credential
   - Credential type: `Zoom OAuth2 API`
   - Response format: `File`
   - Connect `Build File Name` → `Download Recording`

9. **Add a Google Drive node**
   - Node type: `Google Drive`
   - Name it: `Upload to Google Drive`
   - Configure upload/create-file behavior as appropriate in your n8n version
   - Set file name:
     `={{ $('Build File Name').item.json.FileName }}`
   - Drive: `My Drive`
   - Folder ID: set your target Google Drive folder ID
   - Connect `Download Recording` → `Upload to Google Drive`

10. **Configure Google Drive credentials**
    - Use Google Drive OAuth2 credentials in n8n
    - Ensure the authenticated account has write access to the selected folder

11. **Add an HTTP Request node for Gemini file upload**
    - Node type: `HTTP Request`
    - Name it: `Upload Audio to Gemini`
    - Method: `POST`
    - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`
    - Authentication: predefined credential
    - Credential type: `Google PaLM API` / Gemini-compatible credential
    - Enable headers and set:
      - `X-Goog-Upload-Command` = `upload, finalize`
      - `X-Goog-Upload-Protocol` = `raw`
    - Send body: enabled
    - Content type: `Binary Data`
    - Binary property / input data field name: `data`
    - Connect `Download Recording` → `Upload Audio to Gemini`

12. **Configure Gemini credentials**
    - Add your Gemini / Google AI API key in n8n credentials
    - Use the credential type compatible with the Google Generative Language API in your n8n version

13. **Add an HTTP Request node for Gemini analysis**
    - Node type: `HTTP Request`
    - Name it: `Analyze Call with Gemini`
    - Method: `POST`
    - URL:
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
    - Authentication: same Gemini credential
    - Body type: JSON
    - Set the JSON body to send:
      - `contents.parts[0].fileData.mimeType` from `{{$json.file.mimeType}}`
      - `contents.parts[0].fileData.fileUri` from `{{$json.file.uri}}`
      - `contents.parts[1].text` with the analysis instruction
      - `generationConfig.responseMimeType` = `application/json`
      - `generationConfig.responseSchema` defining:
        - `step` as string
        - `advice` as string
        - both required
    - Connect `Upload Audio to Gemini` → `Analyze Call with Gemini`

14. **Use this Gemini prompt logic**
    - In the text instruction, ask Gemini to:
      - analyze the phone call as a sales/telemarketing call
      - determine the stage reached:
        - Contact reached
        - Needs identified
        - Meeting proposed
        - Appointment booked
      - provide specific actionable advice
      - return only valid JSON

15. **Add a Merge node**
    - Node type: `Merge`
    - Name it: `Merge Drive Link + Analysis`
    - Mode: `Combine`
    - Combine by: `Combine All`
    - Connect:
      - `Upload to Google Drive` → Merge input 1
      - `Analyze Call with Gemini` → Merge input 2

16. **Add a Set node to normalize results**
    - Node type: `Set`
    - Name it: `Format Results`
    - Add these fields:
      - `Caller` = `={{ $('Loop Over Items').item.json.caller_name }}`
      - `Date` = `={{ $('Loop Over Items').item.json.date_time }}`
      - `Number` = `={{ $('Loop Over Items').item.json.callee_number }}`
      - `Direction` = `={{ $('Loop Over Items').item.json.direction }}`
      - `Stage` = `={{ JSON.parse($json.candidates[0].content.parts[0].text).step }}`
      - `Advice` = `={{ JSON.parse($json.candidates[0].content.parts[0].text).advice }}`
      - `DriveURL` = `={{ $json.webViewLink }}`
    - Connect `Merge Drive Link + Analysis` → `Format Results`

17. **Add a Google Sheets node**
    - Node type: `Google Sheets`
    - Name it: `Log to Google Sheets`
    - Operation: `Append`
    - Document ID: set your spreadsheet ID
    - Sheet name: `Sheet1` or your target sheet
    - Map columns manually:
      - Caller ← `{{$json.Caller}}`
      - Date ← `{{$json.Date}}`
      - Number ← `{{$json.Number}}`
      - Direction ← `{{$json.Direction}}`
      - Stage ← `{{$json.Stage}}`
      - Advice ← `{{$json.Advice}}`
      - DriveURL ← `{{$json.DriveURL}}`
    - Connect `Format Results` → `Log to Google Sheets`

18. **Prepare the Google Sheet**
    - Create a spreadsheet with at least these columns:
      - Caller
      - Date
      - Number
      - Direction
      - Stage
      - Advice
      - DriveURL

19. **Configure Google Sheets credentials**
    - Use Google Sheets OAuth2 credentials in n8n
    - Ensure edit access to the target spreadsheet

20. **Close the loop**
    - Connect `Log to Google Sheets` → `Loop Over Items`
    - This allows the batch node to continue to the next recording.

21. **Optional but recommended hardening**
    - Add an IF node before processing to skip items with no `download_url`
    - Sanitize filename values to remove problematic characters
    - Add error handling for malformed Gemini output
    - Add deduplication logic so the same recording is not logged multiple times
    - Add pagination support for Zoom if more than 20 recordings are expected per run

22. **Activate the workflow**
    - Test with a small dataset first
    - Confirm:
      - recordings are fetched from Zoom
      - files upload to Drive
      - Gemini returns valid JSON
      - rows append to Sheets correctly

### Sub-workflow setup
This workflow does **not** use sub-workflows or Execute Workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Zoom Server-to-Server OAuth app with `phone:read` and `recording:read` scopes. | Zoom credential preparation |
| Add your Google Gemini API key in n8n credentials. | Gemini authentication |
| Connect Google Drive and Google Sheets OAuth2. | Google service authentication |
| Set the Google Drive folder ID in the `Upload to Google Drive` node. | Required before production use |
| Set the Google Sheets document ID in the `Log to Google Sheets` node. | Required before production use |
| Ensure the sheet contains columns: Caller, Date, Number, Direction, Stage, Advice, Drive URL. | Spreadsheet structure |
| You can change the schedule interval in the trigger node. | Customization |
| You can edit the Gemini prompt to match your own call scoring criteria. | Customization |
| You can add more columns to the spreadsheet for additional tracking fields. | Customization |

## Additional implementation observations
- The workflow currently polls Zoom every hour, but the introductory note says “every few hours”; the actual active configuration is hourly.
- The Zoom API request does not include date filters, so the exact set of returned recordings depends on Zoom’s API defaults.
- There is no deduplication mechanism, so repeated polling may log the same recording multiple times if Zoom keeps returning it.
- The workflow depends on Gemini returning strict JSON inside `candidates[0].content.parts[0].text`; if the API response format changes, `Format Results` will break.
- Both the Google Drive folder ID and Google Sheets document ID are blank in the provided workflow and must be filled manually before use.