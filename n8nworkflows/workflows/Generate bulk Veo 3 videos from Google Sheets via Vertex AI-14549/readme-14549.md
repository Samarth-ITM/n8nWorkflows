Generate bulk Veo 3 videos from Google Sheets via Vertex AI

https://n8nworkflows.xyz/workflows/generate-bulk-veo-3-videos-from-google-sheets-via-vertex-ai-14549


# Generate bulk Veo 3 videos from Google Sheets via Vertex AI

Now I have a complete understanding of the workflow. Let me write the comprehensive documentation.---

### 1. Workflow Overview

This workflow automates **bulk Veo 3 video generation** using Google Vertex AI, orchestrated from a Google Sheets spreadsheet. Each row in the sheet represents one video generation request. When a row is submitted (via a webhook triggered from a Google Apps Script or similar mechanism), the workflow:

1. **Validates** the incoming data (prompt, resolution, aspect ratio, and the `send_for_generation` flag must all be present and correct).
2. **Prepares and sends** an asynchronous video generation request to the Vertex AI `predictLongRunning` endpoint using the Veo 3 model.
3. **Polls** the operation in a retry loop (60-second intervals) until the video is ready or an error is detected.
4. **Post-processes** the resulting Base64-encoded video by converting it to a binary file and uploading it to Google Drive.
5. **Writes back** either the Google Drive video link or an error message into the originating Google Sheet row.

The workflow is designed for unattended bulk processing: each webhook invocation handles one row independently, and multiple rows can be processed concurrently since each execution is isolated.

---

### 2. Block-by-Block Analysis

---

#### Block 1: Trigger and Validation

**Overview:** This block receives an HTTP POST webhook triggered when a Google Sheet row is ready for processing. It validates that all required fields are present and that the `send_for_generation` flag is set to `true`. Rows that fail validation are silently discarded (no false-branch connection exists).

**Nodes Involved:**
- Google Sheet Trigger
- If

**Node Details:**

| Node | Type | Technical Role |
|---|---|---|
| **Google Sheet Trigger** | `n8n-nodes-base.webhook` v2.1 | Entry point — receives POST payloads containing a single sheet row's data. |
| **If** | `n8n-nodes-base.if` v2.2 | Gatekeeper — ensures all mandatory fields exist before proceeding. |

**Google Sheet Trigger**
- **Configuration:** HTTP method POST, custom webhook path `499c5cd9-7fd6-4ddf-829f-3a0eba4b4460`. No authentication on the webhook itself (relies on path obscurity or external auth).
- **Expected payload structure:** `{ body: { data: { send_for_generation, prompt, resolution, aspectRatio, durationSeconds } }, row: <number> }`
- **Input:** External HTTP POST (from Google Apps Script, curl, or another automation).
- **Output:** Passes the raw webhook payload to the **If** node.
- **Edge cases:** If the webhook receives a GET or a malformed JSON body, n8n will return an error. No built-in rate limiting.

**If**
- **Configuration:** Four conditions combined with AND logic (strict type validation, case-sensitive):
  1. `body.data.send_for_generation` must be `true` (boolean check).
  2. `body.data.prompt` must not be empty (string notEmpty).
  3. `body.data.resolution` must not be empty (string notEmpty).
  4. `body.data.aspectRatio` must not be empty (string notEmpty).
- **Expressions used:**
  - `={{ $json.body.data.send_for_generation }}`
  - `={{ $json.body.data.prompt }}`
  - `={{ $json.body.data.resolution }}`
  - `={{ $json.body.data.aspectRatio }}`
- **Input:** Google Sheet Trigger.
- **Output (true):** Data collection. **Output (false):** No connection — the item is silently dropped.
- **Edge cases:** If `send_for_generation` is a string `"true"` instead of boolean `true`, the strict boolean check will fail. If `durationSeconds` is missing or empty, it will pass validation here but may cause issues downstream in the Vertex AI call.

---

#### Block 2: Prepare Video Request

**Overview:** This block extracts and normalizes all necessary parameters from the webhook payload, assigns Vertex AI configuration constants, then fires an asynchronous POST request to the Vertex AI `predictLongRunning` endpoint to start video generation.

**Nodes Involved:**
- Data collection
- Vertex AI Send for Generation

**Node Details:**

| Node | Type | Technical Role |
|---|---|---|
| **Data collection** | `n8n-nodes-base.set` v3.4 | Normalizes and enriches the payload with Vertex AI configuration constants. |
| **Vertex AI Send for Generation** | `n8n-nodes-base.httpRequest` v4.2 | Sends the async video generation request to Vertex AI. |

**Data collection**
- **Configuration:** Ten assignments:
  | Field | Type | Value / Expression |
  |---|---|---|
  | `api_endpoint` | string | `us-central1-aiplatform.googleapis.com` |
  | `project_id` | string | `your_cloud_console_project_ID` (must be replaced by user) |
  | `veo_3_model` | string | `veo-3.0-generate-preview` |
  | `region` | string | `us-central1` |
  | `prompt` | string | `={{ $json.body.data.prompt }}` |
  | `vertex_api` | string | `Vertex-AI-API` (must be replaced with actual API key) |
  | `resolution` | string | `={{ $json.body.data.resolution }}` |
  | `aspectRatio` | string | `={{ $json.body.data.aspectRatio }}` |
  | `durationSeconds` | number | `={{ $json.body.data.durationSeconds }}` |
  | `row` | number | `={{ $json.body.row }}` |
- **Key note:** `vertex_api` is stored as a string value — it is used as a query parameter key in subsequent HTTP requests. The user must replace this placeholder with their actual Vertex AI API key.
- **Input:** If (true branch).
- **Output:** Vertex AI Send for Generation.
- **Edge cases:** If `durationSeconds` is undefined or non-numeric, the downstream JSON body may be malformed. The `project_id` and `vertex_api` placeholders must be updated before deployment.

**Vertex AI Send for Generation**
- **Configuration:**
  - **Method:** POST
  - **URL (expression):** `https://{{ $json.api_endpoint }}/v1/projects/{{ $json.project_id }}/locations/{{ $json.region }}/publishers/google/models/{{ $json.veo_3_model }}:predictLongRunning`
  - **Query parameter:** `key` = `={{ $json.vertex_api }}`
  - **Body (JSON, expression):**
    ```json
    {
      "instances": [{ "prompt": <stringified prompt> }],
      "parameters": {
        "aspectRatio": "<aspectRatio>",
        "compressionQuality": "optimized",
        "durationSeconds": <durationSeconds>,
        "generateAudio": true,
        "resolution": "<resolution>",
        "sampleCount": 1,
        "personGeneration": "allow_adult"
      }
    }
    ```
  - `prompt` is JSON-stringified via `JSON.stringify($json.prompt)`.
  - `generateAudio` is hardcoded to `true`.
  - `personGeneration` is hardcoded to `"allow_adult"`.
  - `sampleCount` is hardcoded to `1`.
  - `compressionQuality` is hardcoded to `"optimized"`.
- **Input:** Data collection.
- **Output:** Wait.
- **Response:** The Vertex AI long-running operation returns an object containing a `name` field (the operation ID) which is used by the polling loop.
- **Edge cases:**
  - Authentication failure if `vertex_api` key is invalid or revoked.
  - 404 if the model name or endpoint is incorrect.
  - Quota exceeded errors from Vertex AI.
  - `durationSeconds` may have a maximum limit imposed by Veo 3 (commonly 8 seconds for preview models).
  - If the `prompt` contains content violating safety policies, the operation may complete but with `raiMediaFilteredCount > 0` later.

---

#### Block 3: Video Generation Loop

**Overview:** After the generation request is submitted, this block polls the Vertex AI operation until the video is either successfully generated or an error/content-filter is detected. It uses two Wait nodes (initial 60-second wait before first check, then 60-second waits between retries) and two If nodes to determine the outcome at each poll.

**Nodes Involved:**
- Wait
- Fetch/Check Video
- If Video Is Created
- If Video Generation Error
- Wait before next video check

**Node Details:**

| Node | Type | Technical Role |
|---|---|---|
| **Wait** | `n8n-nodes-base.wait` v1.1 | Initial delay (60 s) before first poll — gives Vertex AI time to begin processing. |
| **Fetch/Check Video** | `n8n-nodes-base.httpRequest` v4.2 | Polls the `fetchPredictOperation` endpoint to check operation status. |
| **If Video Is Created** | `n8n-nodes-base.if` v2.3 | Checks whether the video was successfully generated (no content-filtered results). |
| **If Video Generation Error** | `n8n-nodes-base.if` v2.3 | Checks whether the operation has errored or content was filtered. |
| **Wait before next video check** | `n8n-nodes-base.wait` v1.1 | Retry delay (60 s) before re-polling if the operation is still in progress. |

**Wait**
- **Configuration:** 60-second wait. Uses webhook resume mechanism (`webhookId: bc232b98-f03d-4768-acb0-a5874a64dd5f`).
- **Input:** Vertex AI Send for Generation.
- **Output:** Fetch/Check Video.
- **Edge cases:** If the n8n instance restarts during a Wait, the execution may be lost unless n8n is configured with a persistent webhook store.

**Fetch/Check Video**
- **Configuration:**
  - **Method:** POST
  - **URL (expression):** `https://{{ $('Data collection').item.json.api_endpoint }}/v1/projects/{{ $('Data collection').item.json.project_id }}/locations/{{ $('Data collection').item.json.region }}/publishers/google/models/{{ $('Data collection').item.json.veo_3_model }}:fetchPredictOperation`
  - **Query parameter:** `key` = `={{ $('Data collection').item.json.vertex_api }}`
  - **Body (JSON, expression):** `{ "operationName": "{{ $json.name }}" }`
  - **Response format:** JSON (explicitly set in options).
- **Important:** This node references `$('Data collection')` to retrieve static config values (api_endpoint, project_id, region, model, vertex_api) rather than relying on the data flowing through the previous Wait node, which only preserves the HTTP response from Vertex AI.
- **Input:** Wait (initial), or Wait before next video check (loop).
- **Output:** If Video Is Created.
- **Edge cases:**
  - If `$json.name` is missing (Vertex AI didn't return an operation name), the request will fail.
  - API key expiry mid-poll.
  - Network timeout if polling takes too long.

**If Video Is Created**
- **Configuration:** Single condition, AND combinator:
  - `$json.response.raiMediaFilteredCount` equals `0` (number comparison).
- **Interpretation:** If `raiMediaFilteredCount == 0`, it means the video was generated successfully without content filtering. If `raiMediaFilteredCount` is undefined or the operation is still running (no `response` field yet), the condition evaluates to false.
- **Input:** Fetch/Check Video.
- **Output (true):** Convert to Binary File (proceeds to post-processing).
- **Output (false):** If Video Generation Error (further diagnosis needed).
- **Edge cases:** If the Vertex AI operation is still in progress, `response.raiMediaFilteredCount` may be undefined/null, causing the strict number comparison to fail — this is intentional: it routes to the error-check node, which in turn routes back to the retry loop.

**If Video Generation Error**
- **Configuration:** Two conditions combined with OR combinator:
  1. `$json.response.raiMediaFilteredCount` is greater than `0` (content was filtered).
  2. `$json.error` exists (object exists check — covers API errors, timeouts, etc.).
- **Input:** If Video Is Created (false branch).
- **Output (true — error detected):** Update Error in Sheet.
- **Output (false — operation still in progress, no error):** Wait before next video check (continue polling).
- **Edge cases:**
  - If both `response.raiMediaFilteredCount > 0` AND `$json.error` exist, the true branch fires and the error is written to the sheet. The loop does not retry in this case.
  - If the operation is still processing and neither condition is met, the false branch triggers another 60-second wait and re-poll.

**Wait before next video check**
- **Configuration:** 60-second wait. Uses webhook resume mechanism (`webhookId: 0e63293c-582f-4e8b-8a4e-ba6d626aaf67`).
- **Input:** If Video Generation Error (false branch).
- **Output:** Fetch/Check Video (loops back).
- **Edge cases:** There is no maximum retry limit built into the workflow. If Vertex AI never completes (e.g., operation stuck), the loop runs indefinitely. Users should consider adding a loop counter or timeout mechanism for production use.

---

#### Block 4: Post-Process Video File

**Overview:** Once the video is successfully generated, the Base64-encoded video data returned by Vertex AI is converted to a binary file and uploaded to a specified Google Drive folder. The file is named with a timestamp for uniqueness.

**Nodes Involved:**
- Convert to Binary File
- Upload Video to Drive

**Node Details:**

| Node | Type | Technical Role |
|---|---|---|
| **Convert to Binary File** | `n8n-nodes-base.convertToFile` v1.1 | Decodes the Base64-encoded video string into a binary file object. |
| **Upload Video to Drive** | `n8n-nodes-base.googleDrive` v3 | Uploads the binary video file to Google Drive. |

**Convert to Binary File**
- **Configuration:**
  - Operation: `toBinary`
  - Source property: `response.videos[0].bytesBase64Encoded` — the Base64-encoded video string from the Vertex AI response.
- **Input:** If Video Is Created (true branch).
- **Output:** Upload Video to Drive.
- **Edge cases:**
  - If `response.videos` is an empty array or the `bytesBase64Encoded` field is missing, the conversion will fail with an error about a missing source property.
  - The node auto-detects the MIME type from the binary data.

**Upload Video to Drive**
- **Configuration:**
  - **File name:** `={{$now.format('yyyy-MM-dd hh:mm:ss')}}` — timestamp-based naming.
  - **Drive:** "My Drive" (selected from list).
  - **Folder ID:** `1zcpqSwOmiheqWowes-kQ6mSPPmUiuSoZ` (folder named "veo").
  - **Credential:** Google Drive OAuth2 API (credential ID: `cVOLv96eiCzjHaUY`, named "Google Drive account").
- **Input:** Convert to Binary File.
- **Output:** Update video Link in sheet.
- **Output data:** Returns the file metadata including `webViewLink` which is used by the next node.
- **Edge cases:**
  - OAuth2 token expiry — requires a valid Google Drive OAuth2 credential with write permissions.
  - Drive storage quota exceeded.
  - If the folder ID becomes invalid (deleted/moved), the upload will fail.
  - File name collisions are unlikely due to timestamp but possible if two uploads happen in the same second.

---

#### Block 5: Update Sheet with Results

**Overview:** This block writes the final outcome back to the originating Google Sheet row — either the Google Drive video link (on success) or the error/reason (on failure). Both update nodes target the same spreadsheet and use `row_number` as the matching column.

**Nodes Involved:**
- Update video Link in sheet
- Update Error in Sheet

**Node Details:**

| Node | Type | Technical Role |
|---|---|---|
| **Update video Link in sheet** | `n8n-nodes-base.googleSheets` v4.7 | Writes the Google Drive `webViewLink` into the `video_drive_link` column. |
| **Update Error in Sheet** | `n8n-nodes-base.googleSheets` v4.7 | Writes the error reason into the `video_drive_link` column. |

**Update video Link in sheet**
- **Configuration:**
  - Operation: `update`
  - **Spreadsheet:** `google_veo_3_bulk_video_generation_template` (ID: `1E3ujHxOFXvokfDnEMK7wcxy21F3YZgKLL_rAIvcZrkI`)
  - **Sheet:** `VEO_3` (gid=0)
  - **Matching column:** `row_number` (value: `={{ $('Data collection').item.json.row }}`)
  - **Columns updated:**
    - `row_number` = `={{ $('Data collection').item.json.row }}`
    - `video_drive_link` = `={{ $json.webViewLink }}`
  - **Credential:** Google Sheets OAuth2 API (credential ID: `pqhCUJdTO98td28J`, named "Google Sheets account").
- **Input:** Upload Video to Drive.
- **Output:** None (end of workflow for success path).
- **Edge cases:**
  - If `row_number` doesn't match an existing row, the update may fail or create unexpected behavior.
  - The `row_number` here refers to the Google Sheets row number, not a sheet-internal ID.

**Update Error in Sheet**
- **Configuration:**
  - Operation: `update`
  - Same spreadsheet and sheet as above.
  - **Matching column:** `row_number` (value: `={{ $('Data collection').item.json.row }}`)
  - **Columns updated:**
    - `row_number` = `={{ $('Data collection').item.json.row }}`
    - `video_drive_link` = `={{ $json.response?.raiMediaFilteredReasons ?? $json.error }}`
  - **Credential:** Same Google Sheets OAuth2 API credential.
- **Input:** If Video Generation Error (true branch).
- **Output:** None (end of workflow for error path).
- **Expression note:** Uses the nullish coalescing operator (`??`) to prefer `raiMediaFilteredReasons` if present, otherwise falls back to `$json.error`. This covers both content-filter scenarios and raw API errors.
- **Edge cases:**
  - `raiMediaFilteredReasons` could be an array or object; it will be serialized as a string when written to the sheet.
  - Same row-number matching caveats as above.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheet Trigger | `n8n-nodes-base.webhook` v2.1 | Receives POST webhook from Google Sheet update | External HTTP POST | If | Trigger workflow from sheet — Starts the workflow on a specified Google Sheet update. |
| If | `n8n-nodes-base.if` v2.2 | Validates required fields before processing | Google Sheet Trigger | Data collection | Bulk Veo 3 Video Generation from Google Sheets via Vertex AI — 1. Triggers a workflow based on a Google Sheet update. 2. Collects data and sends a request to Vertex AI for video generation. 3. Waits and checks if the video is created successfully. 4. Converts the video to a binary file, uploads it to Google Drive, and updates the link in the sheet. 5. Handles errors by updating the error status in the Google Sheet and retrying the video check. |
| Data collection | `n8n-nodes-base.set` v3.4 | Maps webhook data and Vertex AI config constants | If | Vertex AI Send for Generation | Prepare video request — Collects necessary data and sends a POST request to Vertex AI. |
| Vertex AI Send for Generation | `n8n-nodes-base.httpRequest` v4.2 | Sends async video generation request to Vertex AI | Data collection | Wait | Prepare video request — Collects necessary data and sends a POST request to Vertex AI. |
| Wait | `n8n-nodes-base.wait` v1.1 | Initial 60s delay before first poll | Vertex AI Send for Generation | Fetch/Check Video | Video generation loop — Waits for video generation and retries as needed. |
| Fetch/Check Video | `n8n-nodes-base.httpRequest` v4.2 | Polls Vertex AI for operation status | Wait, Wait before next video check | If Video Is Created | Video generation loop — Waits for video generation and retries as needed. |
| If Video Is Created | `n8n-nodes-base.if` v2.3 | Determines if video generation succeeded (no filtered content) | Fetch/Check Video | Convert to Binary File (true), If Video Generation Error (false) | Video generation loop — Waits for video generation and retries as needed. |
| If Video Generation Error | `n8n-nodes-base.if` v2.3 | Determines if error or content filter occurred vs. still processing | If Video Is Created | Update Error in Sheet (true), Wait before next video check (false) | Video generation loop — Waits for video generation and retries as needed. |
| Wait before next video check | `n8n-nodes-base.wait` v1.1 | 60s delay before re-polling Vertex AI | If Video Generation Error | Fetch/Check Video | Video generation loop — Waits for video generation and retries as needed. |
| Convert to Binary File | `n8n-nodes-base.convertToFile` v1.1 | Decodes Base64 video to binary file | If Video Is Created | Upload Video to Drive | Post-process video file — Converts video to a file and uploads it to Google Drive. |
| Upload Video to Drive | `n8n-nodes-base.googleDrive` v3 | Uploads video binary to Google Drive folder | Convert to Binary File | Update video Link in sheet | Post-process video file — Converts video to a file and uploads it to Google Drive. |
| Update video Link in sheet | `n8n-nodes-base.googleSheets` v4.7 | Writes Drive video link back to sheet row | Upload Video to Drive | None | Update sheet with results — Updates Google Sheet with video link or error status. |
| Update Error in Sheet | `n8n-nodes-base.googleSheets` v4.7 | Writes error/reason back to sheet row | If Video Generation Error | None | Update sheet with results — Updates Google Sheet with video link or error status. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add the Webhook Trigger node:**
   - Type: `Webhook` (v2.1)
   - HTTP Method: `POST`
   - Path: Generate a unique path (e.g., a random UUID) or keep the auto-generated one.
   - No authentication required (or configure as needed).
   - Name it: `Google Sheet Trigger`

3. **Add the validation If node:**
   - Type: `If` (v2.2)
   - Conditions (AND combinator, strict type validation):
     - Condition 1: `$json.body.data.send_for_generation` → Boolean → `true`
     - Condition 2: `$json.body.data.prompt` → String → `is not empty`
     - Condition 3: `$json.body.data.resolution` → String → `is not empty`
     - Condition 4: `$json.body.data.aspectRatio` → String → `is not empty`
   - Connect: `Google Sheet Trigger` → `If` (main output to main input).
   - Name it: `If`

4. **Add the Data collection Set node:**
   - Type: `Set` (v3.4)
   - Assignments:
     - `api_endpoint` (string): `us-central1-aiplatform.googleapis.com`
     - `project_id` (string): Replace with your actual Google Cloud project ID.
     - `veo_3_model` (string): `veo-3.0-generate-preview`
     - `region` (string): `us-central1`
     - `prompt` (string): `={{ $json.body.data.prompt }}`
     - `vertex_api` (string): Replace with your actual Vertex AI API key.
     - `resolution` (string): `={{ $json.body.data.resolution }}`
     - `aspectRatio` (string): `={{ $json.body.data.aspectRatio }}`
     - `durationSeconds` (number): `={{ $json.body.data.durationSeconds }}`
     - `row` (number): `={{ $json.body.row }}`
   - Connect: `If` (true output) → `Data collection`.
   - Name it: `Data collection`

5. **Add the Vertex AI Send for Generation HTTP Request node:**
   - Type: `HTTP Request` (v4.2)
   - Method: `POST`
   - URL: `=https://{{ $json.api_endpoint }}/v1/projects/{{ $json.project_id }}/locations/{{ $json.region }}/publishers/google/models/{{ $json.veo_3_model }}:predictLongRunning`
   - Query Parameters: Add one parameter — Name: `key`, Value: `={{ $json.vertex_api }}`
   - Send Body: Yes, specify as JSON.
   - Body (expression):
     ```
     ={
       "instances": [
         {
           "prompt": {{ JSON.stringify($json.prompt) }}
         }
       ],
       "parameters": {
         "aspectRatio": "{{ $json.aspectRatio }}",
         "compressionQuality": "optimized",
         "durationSeconds": {{ $json.durationSeconds }},
         "generateAudio": true,
         "resolution": "{{ $json.resolution }}",
         "sampleCount": 1,
         "personGeneration": "allow_adult"
       }
     }
     ```
   - Connect: `Data collection` → `Vertex AI Send for Generation`.
   - Name it: `Vertex AI Send for Generation`

6. **Add the initial Wait node:**
   - Type: `Wait` (v1.1)
   - Amount: `60` (seconds)
   - Connect: `Vertex AI Send for Generation` → `Wait`.
   - Name it: `Wait`

7. **Add the Fetch/Check Video HTTP Request node:**
   - Type: `HTTP Request` (v4.2)
   - Method: `POST`
   - URL: `=https://{{ $('Data collection').item.json.api_endpoint }}/v1/projects/{{ $('Data collection').item.json.project_id }}/locations/{{ $('Data collection').item.json.region }}/publishers/google/models/{{ $('Data collection').item.json.veo_3_model }}:fetchPredictOperation`
   - Query Parameters: Name: `key`, Value: `={{ $('Data collection').item.json.vertex_api }}`
   - Send Body: Yes, JSON.
   - Body (expression): `={ "operationName": "{{ $json.name }}" }`
   - Options → Response → Response Format: `JSON`
   - Connect: `Wait` → `Fetch/Check Video`.
   - Name it: `Fetch/Check Video`

8. **Add the If Video Is Created node:**
   - Type: `If` (v2.3)
   - Conditions (AND combinator, strict type validation):
     - Condition: `$json.response.raiMediaFilteredCount` → Number → `equals` → `0`
   - Connect: `Fetch/Check Video` → `If Video Is Created`.
   - True output will go to `Convert to Binary File`.
   - False output will go to `If Video Generation Error`.
   - Name it: `If Video Is Created`

9. **Add the If Video Generation Error node:**
   - Type: `If` (v2.3)
   - Conditions (OR combinator, strict type validation):
     - Condition 1: `$json.response.raiMediaFilteredCount` → Number → `greater than` → `0`
     - Condition 2: `$json.error` → Object → `exists`
   - Connect: `If Video Is Created` (false output) → `If Video Generation Error`.
   - True output will go to `Update Error in Sheet`.
   - False output will go to `Wait before next video check`.
   - Name it: `If Video Generation Error`

10. **Add the Wait before next video check node:**
    - Type: `Wait` (v1.1)
    - Amount: `60` (seconds)
    - Connect: `If Video Generation Error` (false output) → `Wait before next video check`.
    - Connect: `Wait before next video check` → `Fetch/Check Video` (this closes the polling loop).
    - Name it: `Wait before next video check`

11. **Add the Convert to Binary File node:**
    - Type: `Convert to File` (v1.1)
    - Operation: `toBinary`
    - Source Property: `response.videos[0].bytesBase64Encoded`
    - Connect: `If Video Is Created` (true output) → `Convert to Binary File`.
    - Name it: `Convert to Binary File`

12. **Add the Upload Video to Drive node:**
    - Type: `Google Drive` (v3)
    - Operation: Upload a file (default).
    - File name: `={{$now.format('yyyy-MM-dd hh:mm:ss')}}`
    - Drive: Select "My Drive" from the list.
    - Folder: Select or enter the target folder ID (e.g., `1zcpqSwOmiheqWowes-kQ6mSPPmUiuSoZ` for a folder named "veo").
    - Credential: Create and assign a **Google Drive OAuth2 API** credential with write access.
    - Connect: `Convert to Binary File` → `Upload Video to Drive`.
    - Name it: `Upload Video to Drive`

13. **Add the Update video Link in sheet node:**
    - Type: `Google Sheets` (v4.7)
    - Operation: `update`
    - Spreadsheet: Select the target spreadsheet (e.g., `google_veo_3_bulk_video_generation_template`).
    - Sheet: Select the target sheet (e.g., `VEO_3`).
    - Mapping Mode: `Define Below`
    - Matching Column: `row_number`
    - Columns to update:
      - `row_number` = `={{ $('Data collection').item.json.row }}`
      - `video_drive_link` = `={{ $json.webViewLink }}`
    - Credential: Create and assign a **Google Sheets OAuth2 API** credential with edit access.
    - Connect: `Upload Video to Drive` → `Update video Link in sheet`.
    - Name it: `Update video Link in sheet`

14. **Add the Update Error in Sheet node:**
    - Type: `Google Sheets` (v4.7)
    - Operation: `update`
    - Same spreadsheet and sheet as above.
    - Mapping Mode: `Define Below`
    - Matching Column: `row_number`
    - Columns to update:
      - `row_number` = `={{ $('Data collection').item.json.row }}`
      - `video_drive_link` = `={{ $json.response?.raiMediaFilteredReasons ?? $json.error }}`
    - Credential: Same Google Sheets OAuth2 API credential as above.
    - Connect: `If Video Generation Error` (true output) → `Update Error in Sheet`.
    - Name it: `Update Error in Sheet`

15. **Configure Google Sheet to trigger the webhook:**
    - In your Google Sheet, use a Google Apps Script or an add-on (e.g., "On edit" trigger) to send a POST request to the webhook URL whenever a row is marked for generation.
    - The POST payload must follow this structure:
      ```json
      {
        "body": {
          "data": {
            "send_for_generation": true,
            "prompt": "A cinematic shot of...",
            "resolution": "1080p",
            "aspectRatio": "16:9",
            "durationSeconds": 8
          }
        },
        "row": 5
      }
      ```
    - The `row` value must correspond to the actual Google Sheets row number so the workflow can write back results.

16. **Verify all connections:**
    - `Google Sheet Trigger` → `If`
    - `If` (true) → `Data collection`
    - `Data collection` → `Vertex AI Send for Generation`
    - `Vertex AI Send for Generation` → `Wait`
    - `Wait` → `Fetch/Check Video`
    - `Fetch/Check Video` → `If Video Is Created`
    - `If Video Is Created` (true) → `Convert to Binary File`
    - `If Video Is Created` (false) → `If Video Generation Error`
    - `If Video Generation Error` (true) → `Update Error in Sheet`
    - `If Video Generation Error` (false) → `Wait before next video check`
    - `Wait before next video check` → `Fetch/Check Video`
    - `Convert to Binary File` → `Upload Video to Drive`
    - `Upload Video to Drive` → `Update video Link in sheet`

17. **Activate the workflow** and test by sending a sample POST to the webhook URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The `vertex_api` field in the Data collection node must contain a valid Google Cloud API key for Vertex AI. Replace the placeholder `Vertex-AI-API` with your actual key before activating the workflow. | Vertex AI API Keys: https://console.cloud.google.com/apis/credentials |
| The `project_id` field must be replaced with your actual Google Cloud project ID where Vertex AI API is enabled. | Google Cloud Console: https://console.cloud.google.com/ |
| The Veo 3 model `veo-3.0-generate-preview` is a preview model — availability, quotas, and pricing may change. Ensure the model is accessible in your project and region (`us-central1`). | Vertex AI Model Garden: https://console.cloud.google.com/vertex-ai/models |
| `durationSeconds` for Veo 3 preview is typically limited to 8 seconds maximum. Submitting a higher value may result in an API error. | Check Vertex AI documentation for current limits. |
| The polling loop has **no built-in retry limit or timeout**. For production use, consider adding a counter (via a Set node that increments a value on each loop iteration) and an If node that stops the workflow after, e.g., 20 iterations (~20 minutes). | n8n community discussion on loop limits. |
| The Google Drive folder ID (`1zcpqSwOmiheqWowes-kQ6mSPPmUiuSoZ`) is hardcoded. If you use a different folder, update the `folderId` parameter in the `Upload Video to Drive` node. | Google Drive folder URL format: `https://drive.google.com/drive/folders/<FOLDER_ID>` |
| The spreadsheet ID (`1E3ujHxOFXvokfDnEMK7wcxy21F3YZgKLL_rAIvcZrkI`) is hardcoded. Duplicate the template sheet or update the `documentId` parameter in both Google Sheets nodes. | Template sheet: https://docs.google.com/spreadsheets/d/1E3ujHxOFXvokfDnEMK7wcxy21F3YZgKLL_rAIvcZrkI/edit |
| The webhook has no authentication. For production deployments, consider adding n8n webhook authentication (Basic Auth, Header Auth, or JWT) to prevent unauthorized triggering. | n8n Webhook Security: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/ |
| The `personGeneration` parameter is set to `allow_adult`. Depending on your use case and Google's usage policies, you may need to change this to `allow_all` or `dont_allow`. | Vertex AI Imagen/Veo safety settings documentation. |
| The `generateAudio` parameter is hardcoded to `true`. Veo 3 supports audio generation natively. If you want silent video, set this to `false`. | Veo 3 API reference. |
| Three OAuth2 credentials are required: Google Drive (write access), Google Sheets (edit access), and a Vertex AI API key (passed as query parameter, not OAuth2). Ensure all are configured and valid before activation. | n8n Credentials documentation: https://docs.n8n.io/credentials/ |
| The `raiMediaFilteredReasons` field from Vertex AI may be an array. When written to a Google Sheets cell, it will appear as a stringified JSON array. Consider formatting it for readability if needed. | Vertex AI Responsible AI documentation. |