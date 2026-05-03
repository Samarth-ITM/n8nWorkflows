Generate UGC product videos from Google Sheets with DALL·E, GPT-4 and Sora

https://n8nworkflows.xyz/workflows/generate-ugc-product-videos-from-google-sheets-with-dall-e--gpt-4-and-sora-14256


# Generate UGC product videos from Google Sheets with DALL·E, GPT-4 and Sora

# 1. Workflow Overview

This workflow automates the generation of short UGC-style product videos from rows stored in Google Sheets. On a schedule, it reads product entries marked as **Pending**, generates a product lifestyle image with **DALL·E 3**, analyzes that image with **GPT-4o vision**, builds a video prompt, submits it to **Sora**, polls until the video is ready, uploads the resulting MP4 to **Google Drive**, makes the file public, and writes the image/video links back to the sheet with **Status = Done**.

The workflow also includes a separate **error-handler workflow**. If the main execution fails, that workflow attempts to recover the processed row number and writes **Status = Error** plus execution details back to the sheet.

## 1.1 Scheduled Intake and Row Selection

This block starts the process on a recurring schedule, reads the spreadsheet, and filters rows to keep only products whose status is **Pending**.

## 1.2 Per-Item Iteration and Image Prompt Generation

This block loops through each pending product row and builds a DALL·E prompt tailored to a UGC/lifestyle aesthetic.

## 1.3 Image Generation and Vision Enrichment

This block generates a product image using DALL·E 3, extracts the returned image URL, and sends that image to GPT-4o for structured visual analysis to inform the later video prompt.

## 1.4 Video Prompt Construction and Sora Job Submission

This block converts the product data and vision analysis into a concise Sora prompt, waits briefly for rate-limit safety, and submits the asynchronous video-generation request.

## 1.5 Sora Polling Loop

This block repeatedly polls the Sora job every 30 seconds until the video is completed or a timeout/error condition is reached.

## 1.6 Video Retrieval, Drive Publication, and Sheet Update

This block downloads the completed video file, uploads it to Google Drive, makes it publicly readable, constructs a shareable URL, updates the source row in Google Sheets, and then returns to the batch loop for the next item.

## 1.7 Error Handling Workflow

This separate workflow is designed to be set as the main workflow’s **Error Workflow** in n8n settings. It parses failed execution details, tries to identify the source spreadsheet row, and writes an error status/message back to Google Sheets when possible.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Intake and Row Selection

**Overview:**  
This block triggers the automation on a time interval, reads all rows from the product sheet, and filters to rows whose `Status` is `Pending`. It also computes a row reference used later for updates.

**Nodes Involved:**  
- Schedule Trigger1  
- Read Product Sheet1  
- Filter Pending Rows1

### Schedule Trigger1
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry point for timed executions.
- **Configuration choices:** Configured with an interval on the `hours` field, meaning the workflow runs hourly by default.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Read Product Sheet1**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** Misconfigured schedule frequency can cause overly frequent OpenAI/API usage or unnecessary idle runs.
- **Sub-workflow reference:** None.

### Read Product Sheet1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads product rows from Google Sheets.
- **Configuration choices:** Reads from spreadsheet `UGC Automation`, sheet `gid=0` / `Sheet1`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Schedule Trigger1**; output to **Filter Pending Rows1**.
- **Version-specific requirements:** Type version `4.5`; requires valid Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:** OAuth token expiry, missing sheet access, renamed sheet, changed column structure, API quota issues.
- **Sub-workflow reference:** None.

### Filter Pending Rows1
- **Type and technical role:** `n8n-nodes-base.code`; transforms raw sheet rows into normalized workflow items.
- **Configuration choices:** Filters only rows where `Status === 'Pending'`. Maps:
  - `Product Name` → `productName`
  - `Description` → `description`
  - `Target Audience` → `targetAudience`
  - plus existing URLs/error fields
  - computes `sheetRowNumber` as `i + 2`
- **Key expressions or variables used:** Internal JavaScript logic; no external expressions.
- **Input and output connections:** Input from **Read Product Sheet1**; output to **Loop Over Items1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Row-number calculation depends on the returned row order matching sheet order.
  - It uses the loop index `i`, not the sheet-provided `row_number`, so filtered/hidden rows or altered read behavior could misalign updates.
  - Only exact `Pending` matches pass; lowercase or trailing variations are ignored except trim.
- **Sub-workflow reference:** None.

---

## 2.2 Per-Item Iteration and Image Prompt Generation

**Overview:**  
This block handles one pending product at a time and constructs a DALL·E-ready image prompt with a lifestyle/UGC visual style.

**Nodes Involved:**  
- Loop Over Items1  
- AI Prompt Builder1  
- Loop Complete1

### Loop Over Items1
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through filtered products one item at a time.
- **Configuration choices:** Default batching behavior; acts as the loop controller.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Filter Pending Rows1** and loopback from **Update Sheet — Done2**. Main item-processing output goes to **AI Prompt Builder1**; completion output goes to **Loop Complete1**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:** If upstream returns no items, the workflow ends naturally. Loop behavior depends on correct return connection from the completion path.
- **Sub-workflow reference:** None.

### AI Prompt Builder1
- **Type and technical role:** `n8n-nodes-base.code`; creates an image-generation prompt.
- **Configuration choices:** Builds a detailed prompt from:
  - `productName`
  - `description`
  - `targetAudience`
  and adds styling directives such as natural lighting, iPhone look, casual setting, authenticity, warm color grading, and no text/logos.
- **Key expressions or variables used:** Internal JS; uses incoming JSON fields.
- **Input and output connections:** Input from **Loop Over Items1**; output to **DALL-E 3 Generate Image1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Missing product fields fall back to defaults like `product` or `young adults`, which may reduce output quality but not break execution.
- **Sub-workflow reference:** None.

### Loop Complete1
- **Type and technical role:** `n8n-nodes-base.noOp`; loop terminator / visual completion marker.
- **Configuration choices:** No operation.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Loop Over Items1** completion output; no output.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

---

## 2.3 Image Generation and Vision Enrichment

**Overview:**  
This block generates a product image via DALL·E 3, extracts the image URL, and obtains structured visual analysis from GPT-4o to improve the downstream video prompt.

**Nodes Involved:**  
- DALL-E 3 Generate Image1  
- Extract DALL-E URL1  
- GPT-4 Vision Analysis1  
- Parse Vision Analysis1

### DALL-E 3 Generate Image1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls OpenAI image generation API.
- **Configuration choices:**  
  - `POST https://api.openai.com/v1/images/generations`
  - Model: `dall-e-3`
  - Prompt: `{{$json.imagePrompt}}`
  - `n: 1`
  - `size: 1024x1024`
  - `quality: hd`
  - `style: natural`
- **Key expressions or variables used:** `{{$json.imagePrompt}}`
- **Input and output connections:** Input from **AI Prompt Builder1**; output to **Extract DALL-E URL1**.
- **Version-specific requirements:** Type version `4.2`; requires OpenAI credentials configured as predefined credential type.
- **Edge cases or potential failure types:** OpenAI auth failure, content-policy rejection, timeout, model availability changes, high latency for image generation.
- **Sub-workflow reference:** None.

### Extract DALL-E URL1
- **Type and technical role:** `n8n-nodes-base.code`; extracts the usable image URL and carries forward previous product data.
- **Configuration choices:** Reads the API response and takes `response.data[0].url` or fallback `revised_url`. Also stores `revised_prompt` if available.
- **Key expressions or variables used:** References prior node data with `$('AI Prompt Builder1').first().json`.
- **Input and output connections:** Input from **DALL-E 3 Generate Image1**; output to **GPT-4 Vision Analysis1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Throws if no image URL is present. This can happen if the OpenAI response structure changes or an upstream error payload is returned unexpectedly.
- **Sub-workflow reference:** None.

### GPT-4 Vision Analysis1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a multimodal chat completion request to analyze the generated image.
- **Configuration choices:**  
  - `POST https://api.openai.com/v1/chat/completions`
  - Model: `gpt-4o`
  - System prompt requests strict JSON with fields:
    - `mood`
    - `dominantColors`
    - `composition`
    - `lighting`
    - `subjectExpression`
    - `settingDetails`
    - `suggestedCameraMovements`
    - `emotionalTone`
  - User message includes product name and the generated `imageUrl`
  - `max_tokens: 1000`
  - `temperature: 0.3`
- **Key expressions or variables used:** Uses `$json.productName` and `$json.imageUrl`.
- **Input and output connections:** Input from **Extract DALL-E URL1**; output to **Parse Vision Analysis1**.
- **Version-specific requirements:** Type version `4.2`; requires OpenAI credentials.
- **Edge cases or potential failure types:** Invalid image URL access, auth issues, timeout, non-JSON or markdown-wrapped JSON output from the model.
- **Sub-workflow reference:** None.

### Parse Vision Analysis1
- **Type and technical role:** `n8n-nodes-base.code`; parses GPT output into structured JSON and provides fallbacks.
- **Configuration choices:** Attempts to parse `choices[0].message.content` as JSON after removing Markdown fences. If parsing fails, it creates a fallback object with generic vision descriptors.
- **Key expressions or variables used:** References **Extract DALL-E URL1** via `$('Extract DALL-E URL1').first().json`.
- **Input and output connections:** Input from **GPT-4 Vision Analysis1**; output to **Video Script Builder1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If OpenAI returns a nonstandard response, parsing may fail; however, the node is resilient because it falls back to default analysis instead of stopping.
- **Sub-workflow reference:** None.

---

## 2.4 Video Prompt Construction and Sora Job Submission

**Overview:**  
This block builds an 8-second vertical-video prompt optimized for Sora, waits to reduce rate-limit pressure, and submits the asynchronous video-generation job.

**Nodes Involved:**  
- Video Script Builder1  
- Wait for Sora Rate Limit1  
- Sora Generate Video1  
- Extract Sora Job ID1

### Video Script Builder1
- **Type and technical role:** `n8n-nodes-base.code`; composes the Sora prompt from product and vision data.
- **Configuration choices:** Produces a concise descriptive prompt including:
  - audience and product usage
  - product description
  - visual style and palette
  - lighting and camera movement
  - authentic user reaction
  - emotional tone
  - “no text overlays or logos”
- **Key expressions or variables used:** Uses fields from `visionAnalysis`, with defaults for missing values.
- **Input and output connections:** Input from **Parse Vision Analysis1**; output to **Wait for Sora Rate Limit1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Low-quality or generic vision data can weaken prompt quality but not break execution.
- **Sub-workflow reference:** None.

### Wait for Sora Rate Limit1
- **Type and technical role:** `n8n-nodes-base.wait`; delay node before Sora submission.
- **Configuration choices:** Waits `60` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Video Script Builder1**; output to **Sora Generate Video1**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Long queues may increase total throughput time; wait-webhook internals require n8n wait support to be functional.
- **Sub-workflow reference:** None.

### Sora Generate Video1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; submits video generation to OpenAI video endpoint.
- **Configuration choices:**  
  - `POST https://api.openai.com/v1/videos`
  - JSON body:
    - `model: "sora-2"`
    - `prompt: $json.videoScript`
    - `seconds: "8"`
    - `size: "720x1280"`
  - Timeout: `300000 ms`
- **Key expressions or variables used:** `$json.videoScript`
- **Input and output connections:** Input from **Wait for Sora Rate Limit1**; output to **Extract Sora Job ID1**.
- **Version-specific requirements:** Type version `4.2`; uses OpenAI credential named `OpenAi account`.
- **Edge cases or potential failure types:** Model access restrictions, endpoint changes, auth failure, long queue times, unsupported account permissions, payload validation errors.
- **Sub-workflow reference:** None.

### Extract Sora Job ID1
- **Type and technical role:** `n8n-nodes-base.code`; validates the Sora submission response and extracts the async job ID.
- **Configuration choices:** Pulls `response.id`, throws if missing, and carries forward product/sheet/image data plus initial poll counters.
- **Key expressions or variables used:** References **Video Script Builder1** data via `$('Video Script Builder1').first().json`.
- **Input and output connections:** Input from **Sora Generate Video1**; output to **Wait 30 Seconds1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Throws on malformed or unexpected Sora response; if the API returns a queued object without `id`, the workflow stops and the error workflow should catch it.
- **Sub-workflow reference:** None.

---

## 2.5 Sora Polling Loop

**Overview:**  
This block polls the async Sora job every 30 seconds until it completes. It also enforces a maximum of 40 polls as a 20-minute timeout safeguard.

**Nodes Involved:**  
- Wait 30 Seconds1  
- Poll Sora Status1  
- Check Sora Status1  
- Video Ready?1

### Wait 30 Seconds1
- **Type and technical role:** `n8n-nodes-base.wait`; delay between Sora status checks.
- **Configuration choices:** Waits `30` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Extract Sora Job ID1** and the false branch of **Video Ready?1**; output to **Poll Sora Status1**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Depends on n8n wait execution persistence; if disabled, long-running async flows may not behave correctly.
- **Sub-workflow reference:** None.

### Poll Sora Status1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; polls job status.
- **Configuration choices:**  
  - `GET https://api.openai.com/v1/videos/{{ $json.videoId }}`
  - Timeout: `30000 ms`
- **Key expressions or variables used:** `{{$json.videoId}}`
- **Input and output connections:** Input from **Wait 30 Seconds1**; output to **Check Sora Status1**.
- **Version-specific requirements:** Type version `4.2`; requires OpenAI credentials.
- **Edge cases or potential failure types:** Network issues, auth failures, unknown `videoId`, status schema changes.
- **Sub-workflow reference:** None.

### Check Sora Status1
- **Type and technical role:** `n8n-nodes-base.code`; merges polled status with original product context and enforces timeout/error rules.
- **Configuration choices:**  
  - Reads status and progress
  - Retrieves carried metadata from **Extract Sora Job ID1**
  - Increments `pollCount`
  - Throws if response contains `error`
  - Throws if `pollCount >= 40`
  - Sets `isDone = status === 'completed'`
- **Key expressions or variables used:** `$('Extract Sora Job ID1').first().json`
- **Input and output connections:** Input from **Poll Sora Status1**; output to **Video Ready?1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Because it always references the first item from **Extract Sora Job ID1**, it assumes one active item context at a time; this works with the current sequential loop design.
  - Throws on explicit API errors or timeout.
- **Sub-workflow reference:** None.

### Video Ready?1
- **Type and technical role:** `n8n-nodes-base.if`; branching node to continue polling or move to file retrieval.
- **Configuration choices:** Checks whether `{{$json.isDone}} === true`.
- **Key expressions or variables used:** `{{$json.isDone}}`
- **Input and output connections:** Input from **Check Sora Status1**. True branch goes to **Fetch Sora Video Content1**; false branch goes back to **Wait 30 Seconds1**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** If the status field schema changes and `isDone` is never set true, the workflow will eventually hit the timeout in **Check Sora Status1**.
- **Sub-workflow reference:** None.

---

## 2.6 Video Retrieval, Drive Publication, and Sheet Update

**Overview:**  
After Sora completes, this block downloads the MP4, uploads it to Drive, makes it public, creates a shareable link, updates the corresponding sheet row, and returns to the loop controller.

**Nodes Involved:**  
- Fetch Sora Video Content1  
- Upload Video to Google Drive1  
- Make File Public1  
- Build Public Drive URL1  
- Update Sheet — Done2

### Fetch Sora Video Content1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the generated video file as binary.
- **Configuration choices:**  
  - `GET https://api.openai.com/v1/videos/{{ $json.videoId }}/content`
  - Response format set to `file`
  - Timeout: `120000 ms`
- **Key expressions or variables used:** `{{$json.videoId}}`
- **Input and output connections:** Input from **Video Ready?1** true branch; output to **Upload Video to Google Drive1**.
- **Version-specific requirements:** Type version `4.2`; requires OpenAI credentials and binary mode support.
- **Edge cases or potential failure types:** File download timeout, unavailable content endpoint, binary handling issues, large-file transfer interruptions.
- **Sub-workflow reference:** None.

### Upload Video to Google Drive1
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the binary MP4 to a target Drive folder.
- **Configuration choices:**  
  - File name expression:
    `{{ $('Check Sora Status1').first().json.productName || 'ugc-video' }}_{{ $('Check Sora Status1').first().json.videoId }}.mp4`
  - Drive: `My Drive`
  - Folder: `UGC Videos`
- **Key expressions or variables used:** Reads `productName` and `videoId` from **Check Sora Status1**.
- **Input and output connections:** Input from **Fetch Sora Video Content1**; output to **Make File Public1**.
- **Version-specific requirements:** Type version `3`; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:** Missing binary data, folder permission issues, expired OAuth token, invalid folder ID.
- **Sub-workflow reference:** None.

### Make File Public1
- **Type and technical role:** `n8n-nodes-base.httpRequest`; creates a public “anyone can read” permission on the uploaded Drive file.
- **Configuration choices:**  
  - `POST https://www.googleapis.com/drive/v3/files/{{ $json.id }}/permissions`
  - JSON body:
    - `role: reader`
    - `type: anyone`
  - Uses Google Drive OAuth2 credentials
- **Key expressions or variables used:** `{{$json.id}}`
- **Input and output connections:** Input from **Upload Video to Google Drive1**; output to **Build Public Drive URL1**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:** Drive API permission restrictions, domain policies disallowing public sharing, auth errors, duplicate/ignored permission calls.
- **Sub-workflow reference:** None.

### Build Public Drive URL1
- **Type and technical role:** `n8n-nodes-base.code`; constructs the final public video URL and restores original product metadata.
- **Configuration choices:** Reads uploaded file ID from **Upload Video to Google Drive1** and context data from **Check Sora Status1**. Builds:
  `https://drive.google.com/file/d/{fileId}/view?usp=sharing`
- **Key expressions or variables used:** Uses `$('Upload Video to Google Drive1').first().json` and `$('Check Sora Status1').first().json`.
- **Input and output connections:** Input from **Make File Public1**; output to **Update Sheet — Done2**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If upload response lacks `id`, URL generation fails or becomes invalid.
- **Sub-workflow reference:** None.

### Update Sheet — Done2
- **Type and technical role:** `n8n-nodes-base.googleSheets`; writes final results back to the source row.
- **Configuration choices:** Updates matching row by `row_number` with:
  - `Status = Done`
  - `Image URL`
  - `Video URL`
  - `Description`
  - `Product Name`
  - `Target Audience`
  - `row_number = {{$json.sheetRowNumber}}`
- **Key expressions or variables used:** Uses current JSON fields such as `imageUrl`, `videoUrl`, `sheetRowNumber`.
- **Input and output connections:** Input from **Build Public Drive URL1**; output loops back to **Loop Over Items1**.
- **Version-specific requirements:** Type version `4.5`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Matching depends on the `row_number` column existing and aligning with `sheetRowNumber`.
  - The workflow reads from `Sheet1/gid=0`, while the error handler writes to a sheet named `Products`; this mismatch may cause inconsistent updates unless both point to the intended tab.
- **Sub-workflow reference:** None.

---

## 2.7 Error Handling Workflow

**Overview:**  
This is a separate workflow intended to capture failures from the main workflow. It extracts execution metadata, attempts to find the sheet row being processed, and either updates the relevant row in Google Sheets or logs the error when the row cannot be identified.

**Nodes Involved:**  
- Error Trigger  
- Parse Error Details  
- Row Number Known?  
- Update Sheet — Error  
- Log Error (Unknown Row)

### Error Trigger
- **Type and technical role:** `n8n-nodes-base.errorTrigger`; entry point for workflow-failure events.
- **Configuration choices:** No additional parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; output to **Parse Error Details**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Only runs if this workflow is configured as the **Error Workflow** for the main workflow or relevant executions.
- **Sub-workflow reference:** Separate error workflow for the main UGC pipeline.

### Parse Error Details
- **Type and technical role:** `n8n-nodes-base.code`; parses failed execution payload.
- **Configuration choices:**  
  - Reads `execution.error.message`
  - Falls back to `execution.lastNodeExecuted`
  - Traverses `execution.data.resultData.runData` in reverse order
  - Attempts to locate an item containing `sheetRowNumber`
  - Returns:
    - `sheetRowNumber`
    - `productName`
    - `errorMessage` truncated to 500 chars
    - `failedNode`
    - `executionId`
    - `timestamp`
- **Key expressions or variables used:** Internal JS on incoming execution payload.
- **Input and output connections:** Input from **Error Trigger**; output to **Row Number Known?**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Execution payload shape may vary by n8n version; row extraction may fail if no downstream item carried `sheetRowNumber`.
- **Sub-workflow reference:** Error workflow internal logic.

### Row Number Known?
- **Type and technical role:** `n8n-nodes-base.if`; branches depending on whether `sheetRowNumber` exists.
- **Configuration choices:** Uses the `exists` operator on `{{$json.sheetRowNumber}}`.
- **Key expressions or variables used:** `{{$json.sheetRowNumber}}`
- **Input and output connections:** Input from **Parse Error Details**. True branch to **Update Sheet — Error**; false branch to **Log Error (Unknown Row)**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:** If row number is present but incorrect, the wrong sheet row could be updated.
- **Sub-workflow reference:** Error workflow internal logic.

### Update Sheet — Error
- **Type and technical role:** `n8n-nodes-base.googleSheets`; writes error details back to the spreadsheet.
- **Configuration choices:** Updates sheet named `Products` in spreadsheet `UGC Automation`, setting:
  - `Status = Error`
  - `Error Message = errorMessage + failedNode + executionId`
  - Match column: `row_number`
- **Key expressions or variables used:** Uses `{{$json.errorMessage}}`, `{{$json.failedNode}}`, `{{$json.executionId}}`
- **Input and output connections:** Input from **Row Number Known?** true branch; no output.
- **Version-specific requirements:** Type version `4.5`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Possible sheet-name mismatch with the main workflow (`Products` vs `Sheet1/gid=0`)
  - Matching by `row_number` requires that column to exist and hold expected values
  - If `row_number` is not passed/mapped correctly, update may fail silently or target no row
- **Sub-workflow reference:** Error workflow internal logic.

### Log Error (Unknown Row)
- **Type and technical role:** `n8n-nodes-base.code`; console-logs errors that cannot be tied to a row.
- **Configuration choices:** Uses `console.warn` and returns the same item plus `logged: true`.
- **Key expressions or variables used:** None beyond incoming JSON.
- **Input and output connections:** Input from **Row Number Known?** false branch; no output.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Logging works only if execution logs are retained and accessible.
- **Sub-workflow reference:** Error workflow internal logic.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Error Trigger | Error Trigger | Starts the error workflow when another workflow execution fails |  | Parse Error Details | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Parse Error Details | Code | Extracts row number, product name, failed node, execution ID, and message from the failed execution | Error Trigger | Row Number Known? | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Row Number Known? | If | Checks whether the failed item can be mapped back to a sheet row | Parse Error Details | Update Sheet — Error; Log Error (Unknown Row) | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Update Sheet — Error | Google Sheets | Writes Status = Error and error details back to the sheet | Row Number Known? |  | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Log Error (Unknown Row) | Code | Logs an error when the source row cannot be identified | Row Number Known? |  | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Sticky Note | Sticky Note | Visual documentation for error workflow |  |  | ## ERROR HANDLER WORKFLOW<br>This workflow is triggered when the main UGC pipeline fails.<br>It extracts the error details and row number, then updates<br>the Google Sheet with Status = "Error" and the error message.<br><br>SETUP: In the main workflow settings, set this workflow<br>as the "Error Workflow". |
| Sticky Note — Complete1 | Sticky Note | Visual documentation for final upload and sheet update block |  |  |  |
| Sticky Note — Sora Poll Loop1 | Sticky Note | Visual documentation for polling loop |  |  |  |
| Sticky Note — Sora Async Polling1 | Sticky Note | Visual documentation for Sora async generation block |  |  |  |
| Sticky Note — Vision & Script1 | Sticky Note | Visual documentation for vision analysis and script block |  |  |  |
| Sticky Note — Image Gen1 | Sticky Note | Visual documentation for image generation block |  |  |  |
| Sticky Note — Trigger1 | Sticky Note | Visual documentation for trigger/data-fetch block |  |  |  |
| Loop Complete1 | No Operation, do nothing | Terminates the split-in-batches loop when all items are processed | Loop Over Items1 |  |  |
| Update Sheet — Done2 | Google Sheets | Writes final image/video URLs and marks the row Done | Build Public Drive URL1 | Loop Over Items1 | ## 5. GOOGLE DRIVE UPLOAD & SHEET UPDATE<br>Fetches the video binary from Sora API.<br>Uploads MP4 to Google Drive.<br>Sets file to public (anyone with link).<br>Builds shareable Drive URL.<br>Writes Image URL + Video URL back to the<br>correct row. Sets Status = "Done".<br>Loops back for next pending product.<br>On error, the Error Handler workflow sets<br>Status = "Error" + error message. |
| Build Public Drive URL1 | Code | Builds the public Drive file URL and restores carried item metadata | Make File Public1 | Update Sheet — Done2 | ## 5. GOOGLE DRIVE UPLOAD & SHEET UPDATE<br>Fetches the video binary from Sora API.<br>Uploads MP4 to Google Drive.<br>Sets file to public (anyone with link).<br>Builds shareable Drive URL.<br>Writes Image URL + Video URL back to the<br>correct row. Sets Status = "Done".<br>Loops back for next pending product.<br>On error, the Error Handler workflow sets<br>Status = "Error" + error message. |
| Make File Public1 | HTTP Request | Adds a public-read permission to the uploaded Drive file | Upload Video to Google Drive1 | Build Public Drive URL1 | ## 5. GOOGLE DRIVE UPLOAD & SHEET UPDATE<br>Fetches the video binary from Sora API.<br>Uploads MP4 to Google Drive.<br>Sets file to public (anyone with link).<br>Builds shareable Drive URL.<br>Writes Image URL + Video URL back to the<br>correct row. Sets Status = "Done".<br>Loops back for next pending product.<br>On error, the Error Handler workflow sets<br>Status = "Error" + error message. |
| Upload Video to Google Drive1 | Google Drive | Uploads the generated MP4 into a Drive folder | Fetch Sora Video Content1 | Make File Public1 | ## 5. GOOGLE DRIVE UPLOAD & SHEET UPDATE<br>Fetches the video binary from Sora API.<br>Uploads MP4 to Google Drive.<br>Sets file to public (anyone with link).<br>Builds shareable Drive URL.<br>Writes Image URL + Video URL back to the<br>correct row. Sets Status = "Done".<br>Loops back for next pending product.<br>On error, the Error Handler workflow sets<br>Status = "Error" + error message. |
| Fetch Sora Video Content1 | HTTP Request | Downloads the completed video file from the OpenAI video endpoint | Video Ready?1 | Upload Video to Google Drive1 | ## 5. GOOGLE DRIVE UPLOAD & SHEET UPDATE<br>Fetches the video binary from Sora API.<br>Uploads MP4 to Google Drive.<br>Sets file to public (anyone with link).<br>Builds shareable Drive URL.<br>Writes Image URL + Video URL back to the<br>correct row. Sets Status = "Done".<br>Loops back for next pending product.<br>On error, the Error Handler workflow sets<br>Status = "Error" + error message. |
| Video Ready?1 | If | Branches between polling again and fetching the completed video | Check Sora Status1 | Fetch Sora Video Content1; Wait 30 Seconds1 | ## 4b. SORA POLLING LOOP<br>Polls GET /v1/videos/{videoId} every 30s. Check Sora Status merges poll response with<br>carried product data from Extract Sora Job ID.<br><br>IF node checks isDone === true:<br>  TRUE  -> Fetch video content + upload to Drive<br>  FALSE -> Loop back to Wait 30 Seconds<br>Max 40 polls = 20 minute timeout safety. |
| Check Sora Status1 | Code | Merges Sora status with product context and enforces timeout/error handling | Poll Sora Status1 | Video Ready?1 | ## 4b. SORA POLLING LOOP<br>Polls GET /v1/videos/{videoId} every 30s. Check Sora Status merges poll response with<br>carried product data from Extract Sora Job ID.<br><br>IF node checks isDone === true:<br>  TRUE  -> Fetch video content + upload to Drive<br>  FALSE -> Loop back to Wait 30 Seconds<br>Max 40 polls = 20 minute timeout safety. |
| Poll Sora Status1 | HTTP Request | Checks current status of the Sora video job | Wait 30 Seconds1 | Check Sora Status1 | ## 4b. SORA POLLING LOOP<br>Polls GET /v1/videos/{videoId} every 30s. Check Sora Status merges poll response with<br>carried product data from Extract Sora Job ID.<br><br>IF node checks isDone === true:<br>  TRUE  -> Fetch video content + upload to Drive<br>  FALSE -> Loop back to Wait 30 Seconds<br>Max 40 polls = 20 minute timeout safety. |
| Wait 30 Seconds1 | Wait | Delays each Sora status check by 30 seconds | Extract Sora Job ID1; Video Ready?1 | Poll Sora Status1 | ## 4b. SORA POLLING LOOP<br>Polls GET /v1/videos/{videoId} every 30s. Check Sora Status merges poll response with<br>carried product data from Extract Sora Job ID.<br><br>IF node checks isDone === true:<br>  TRUE  -> Fetch video content + upload to Drive<br>  FALSE -> Loop back to Wait 30 Seconds<br>Max 40 polls = 20 minute timeout safety. |
| Extract Sora Job ID1 | Code | Extracts the async video job ID from the Sora submission response | Sora Generate Video1 | Wait 30 Seconds1 | ## 4. SORA VIDEO GENERATION (Async + Polling)<br>Waits 60s for rate limiting, then POSTs to Sora API. Sora returns a queued job ID (not a video URL).<br><br>Extracts the job ID, then enters a polling loop:<br>  - Wait 30s between polls<br>  - GET /v1/videos/{id} to check status<br>  - If status != "completed", loop back to wait<br>  - Max 40 polls (20 min timeout)<br>Once completed, fetches video content URL. |
| Sora Generate Video1 | HTTP Request | Submits the video-generation request to Sora | Wait for Sora Rate Limit1 | Extract Sora Job ID1 | ## 4. SORA VIDEO GENERATION (Async + Polling)<br>Waits 60s for rate limiting, then POSTs to Sora API. Sora returns a queued job ID (not a video URL).<br><br>Extracts the job ID, then enters a polling loop:<br>  - Wait 30s between polls<br>  - GET /v1/videos/{id} to check status<br>  - If status != "completed", loop back to wait<br>  - Max 40 polls (20 min timeout)<br>Once completed, fetches video content URL. |
| Wait for Sora Rate Limit1 | Wait | Adds a 60-second delay before Sora submission | Video Script Builder1 | Sora Generate Video1 | ## 4. SORA VIDEO GENERATION (Async + Polling)<br>Waits 60s for rate limiting, then POSTs to Sora API. Sora returns a queued job ID (not a video URL).<br><br>Extracts the job ID, then enters a polling loop:<br>  - Wait 30s between polls<br>  - GET /v1/videos/{id} to check status<br>  - If status != "completed", loop back to wait<br>  - Max 40 polls (20 min timeout)<br>Once completed, fetches video content URL. |
| Video Script Builder1 | Code | Builds a Sora-ready UGC video prompt from product and vision data | Parse Vision Analysis1 | Wait for Sora Rate Limit1 | ## 3. VISION ANALYSIS & VIDEO SCRIPT<br>GPT-4 Vision analyzes the generated image for mood, colors, composition. This enriches the video script with visual continuity. Script optimized for Sora's prompt format. |
| Parse Vision Analysis1 | Code | Parses structured vision-analysis output or falls back to defaults | GPT-4 Vision Analysis1 | Video Script Builder1 | ## 3. VISION ANALYSIS & VIDEO SCRIPT<br>GPT-4 Vision analyzes the generated image for mood, colors, composition. This enriches the video script with visual continuity. Script optimized for Sora's prompt format. |
| GPT-4 Vision Analysis1 | HTTP Request | Analyzes the generated image with GPT-4o and returns visual descriptors | Extract DALL-E URL1 | Parse Vision Analysis1 | ## 3. VISION ANALYSIS & VIDEO SCRIPT<br>GPT-4 Vision analyzes the generated image for mood, colors, composition. This enriches the video script with visual continuity. Script optimized for Sora's prompt format. |
| Extract DALL-E URL1 | Code | Extracts the generated image URL from the DALL·E response | DALL-E 3 Generate Image1 | GPT-4 Vision Analysis1 | ## 2. IMAGE GENERATION<br><br>For each pending product, a code node builds a detailed DALL·E 3 prompt with UGC aesthetic direction. The prompt is sent to the OpenAI API to generate an HD image, and the returned image URL is extracted for downstream use. |
| DALL-E 3 Generate Image1 | HTTP Request | Generates a product image with DALL·E 3 | AI Prompt Builder1 | Extract DALL-E URL1 | ## 2. IMAGE GENERATION<br><br>For each pending product, a code node builds a detailed DALL·E 3 prompt with UGC aesthetic direction. The prompt is sent to the OpenAI API to generate an HD image, and the returned image URL is extracted for downstream use. |
| AI Prompt Builder1 | Code | Builds a UGC/lifestyle image prompt from the sheet data | Loop Over Items1 | DALL-E 3 Generate Image1 | ## 2. IMAGE GENERATION<br><br>For each pending product, a code node builds a detailed DALL·E 3 prompt with UGC aesthetic direction. The prompt is sent to the OpenAI API to generate an HD image, and the returned image URL is extracted for downstream use. |
| Loop Over Items1 | Split In Batches | Iterates through pending products one at a time | Filter Pending Rows1; Update Sheet — Done2 | Loop Complete1; AI Prompt Builder1 |  |
| Filter Pending Rows1 | Code | Filters sheet rows to Status = Pending and maps fields into workflow JSON | Read Product Sheet1 | Loop Over Items1 | ## Stage 1 — Trigger & data fetch. <br><br>A schedule trigger fires on a set interval. It reads all rows from a Google Sheets "Products" tab, then a code node filters down to only rows where Status = "Pending", tracking each row number for later updates. |
| Read Product Sheet1 | Google Sheets | Reads all source product rows from the spreadsheet | Schedule Trigger1 | Filter Pending Rows1 | ## Stage 1 — Trigger & data fetch. <br><br>A schedule trigger fires on a set interval. It reads all rows from a Google Sheets "Products" tab, then a code node filters down to only rows where Status = "Pending", tracking each row number for later updates. |
| Schedule Trigger1 | Schedule Trigger | Starts the main workflow on an hourly interval |  | Read Product Sheet1 | ## Stage 1 — Trigger & data fetch. <br><br>A schedule trigger fires on a set interval. It reads all rows from a Google Sheets "Products" tab, then a code node filters down to only rows where Status = "Pending", tracking each row number for later updates. |
| Sticky Note1 | Sticky Note | General overview note |  |  | ## Workflow overview<br><br>This is an n8n automation workflow that turns product listings in a Google Sheet into UGC-style videos, fully hands-off.<br><br>It runs on a schedule, picks up any row marked "Pending" in the sheet, and for each product it: generates a product image with DALL·E 3, analyzes that image with GPT-4 Vision to inform the video style, sends a tailored prompt to Sora to generate a short video, uploads the finished video to Google Drive, and writes the public links back to the sheet. The row's status gets updated to "Done" when complete, or "Error" with details if something fails along the way.<br><br>In short: you fill in product info in a spreadsheet, and the workflow automatically produces and delivers AI-generated marketing videos for each one. |
| Sticky Note6 | Sticky Note | Setup instructions note |  |  | ## Setup instructions<br><br>Before running this workflow, configure the following:<br><br>1. **Google Sheets OAuth** — Connect your Google account so the workflow can read product rows and write back status updates, image URLs, and video URLs.<br>2. **Google Drive credentials** — Needed for uploading finished MP4 videos and setting them to public access.<br>3. **OpenAI API key** — Used for both DALL·E 3 image generation and GPT-4 Vision analysis. Set this in the HTTP Request nodes for those calls.<br>4. **Sora API key** — Used for video generation. Configure in the Sora Generate Video and Poll Sora Status HTTP Request nodes.<br>5. **Google Sheet setup** — Create a sheet named "Products" with columns for product info, Status, Image URL, Video URL, and Error Message. Mark rows you want processed with Status = "Pending".<br>6. **Google Drive folder** — Set the target folder ID in the Upload Video node where finished videos will be stored.<br>7. **Schedule interval** — Adjust the Schedule Trigger to your preferred frequency (e.g. every 15 minutes, hourly).<br>8. **Error handler workflow** — Import the error handler as a separate workflow and link it in this workflow's settings under "Error Workflow" so failed rows get logged back to the sheet. |

---

# 4. Reproducing the Workflow from Scratch

## A. Prepare external services

1. **Create the Google Sheet**
   - Create a spreadsheet, e.g. `UGC Automation`.
   - Add one worksheet for the main pipeline.
   - Recommended columns:
     - `Product Name`
     - `Description`
     - `Target Audience`
     - `Status`
     - `Image URL`
     - `Video URL`
     - `Error Message`
     - `row_number` if you want explicit matching support
   - Mark rows to process with `Status = Pending`.

2. **Create the Google Drive destination**
   - Create a folder for generated videos, e.g. `UGC Videos`.
   - Copy the folder ID.

3. **Configure credentials in n8n**
   - **Google Sheets OAuth2**
   - **Google Drive OAuth2**
   - **OpenAI API**
   - If your environment distinguishes Sora access separately, ensure the same OpenAI credential has video endpoint permission.

---

## B. Build the main workflow

### Stage 1: Trigger and fetch rows

4. **Add `Schedule Trigger`**
   - Name: `Schedule Trigger1`
   - Set interval to hourly, or your preferred cadence.

5. **Add `Google Sheets` node**
   - Name: `Read Product Sheet1`
   - Connect from `Schedule Trigger1`
   - Operation: read/get rows
   - Select your spreadsheet
   - Select the main products tab (`Sheet1` / `gid=0` in the original workflow)

6. **Add `Code` node**
   - Name: `Filter Pending Rows1`
   - Connect from `Read Product Sheet1`
   - Paste logic that:
     - loops through all incoming rows
     - keeps only rows where `Status` equals `Pending`
     - maps columns into JSON keys:
       - `productName`
       - `description`
       - `targetAudience`
       - `status`
       - `imageUrl`
       - `videoUrl`
       - `errorMessage`
     - computes `sheetRowNumber`
   - In the original workflow, `sheetRowNumber` is set to `i + 2`.

   Suggested logic behavior:
   - Return an empty array if no pending rows exist so execution ends cleanly.

### Stage 2: Loop and generate image prompt

7. **Add `Split In Batches`**
   - Name: `Loop Over Items1`
   - Connect from `Filter Pending Rows1`
   - Keep default settings for single-item iteration.

8. **Add `No Operation`**
   - Name: `Loop Complete1`
   - Connect the first output of `Loop Over Items1` to this node as the loop completion branch.

9. **Add `Code` node**
   - Name: `AI Prompt Builder1`
   - Connect the processing output of `Loop Over Items1` to this node.
   - Build an `imagePrompt` string from:
     - product name
     - description
     - target audience
     - UGC aesthetic instructions
   - Include constraints like:
     - natural lighting
     - casual setting
     - photorealistic
     - no logos/text/watermarks

### Stage 3: Generate image and analyze it

10. **Add `HTTP Request`**
    - Name: `DALL-E 3 Generate Image1`
    - Connect from `AI Prompt Builder1`
    - Method: `POST`
    - URL: `https://api.openai.com/v1/images/generations`
    - Authentication: predefined credential type
    - Credential: OpenAI
    - Send JSON body:
      - `model: "dall-e-3"`
      - `prompt: {{$json.imagePrompt}}`
      - `n: 1`
      - `size: "1024x1024"`
      - `quality: "hd"`
      - `style: "natural"`
    - Timeout: about 120000 ms

11. **Add `Code` node**
    - Name: `Extract DALL-E URL1`
    - Connect from `DALL-E 3 Generate Image1`
    - Extract `data[0].url`
    - If missing, optionally try fallback fields
    - Throw an error if no image URL exists
    - Carry forward prior product fields and save:
      - `imageUrl`
      - optionally `dalleRevisedPrompt`

12. **Add `HTTP Request`**
    - Name: `GPT-4 Vision Analysis1`
    - Connect from `Extract DALL-E URL1`
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: OpenAI credential
    - Body should call `gpt-4o` with:
      - a system message asking for JSON only
      - a user message containing:
        - product name text
        - the generated image URL as an `image_url` content block
    - Set `max_tokens` around 1000
    - Set low temperature, e.g. `0.3`

13. **Add `Code` node**
    - Name: `Parse Vision Analysis1`
    - Connect from `GPT-4 Vision Analysis1`
    - Parse `choices[0].message.content`
    - Remove markdown code fences if needed
    - Convert to JSON
    - If parsing fails, create a fallback `visionAnalysis` object with defaults such as:
      - mood
      - dominantColors
      - lighting
      - settingDetails
      - suggestedCameraMovements
      - emotionalTone

### Stage 4: Build Sora prompt and submit async generation

14. **Add `Code` node**
    - Name: `Video Script Builder1`
    - Connect from `Parse Vision Analysis1`
    - Build a concise `videoScript` prompt optimized for Sora:
      - UGC smartphone style
      - audience using the product
      - mood, colors, lighting
      - camera movement
      - no text/logos
      - 8-second vertical short-form feel

15. **Add `Wait` node**
    - Name: `Wait for Sora Rate Limit1`
    - Connect from `Video Script Builder1`
    - Wait amount: `60` seconds

16. **Add `HTTP Request`**
    - Name: `Sora Generate Video1`
    - Connect from `Wait for Sora Rate Limit1`
    - Method: `POST`
    - URL: `https://api.openai.com/v1/videos`
    - Authentication: OpenAI credential
    - Send JSON body:
      - `model: "sora-2"`
      - `prompt: {{$json.videoScript}}`
      - `seconds: "8"`
      - `size: "720x1280"`
    - Timeout: about 300000 ms

17. **Add `Code` node**
    - Name: `Extract Sora Job ID1`
    - Connect from `Sora Generate Video1`
    - Extract response `id` as `videoId`
    - Throw an error if the ID is missing
    - Carry forward:
      - `productName`
      - `description`
      - `targetAudience`
      - `sheetRowNumber`
      - `imageUrl`
      - `videoId`
      - `videoStatus`
      - `progress`
      - `pollCount = 0`

### Stage 5: Poll Sora until complete

18. **Add `Wait` node**
    - Name: `Wait 30 Seconds1`
    - Connect from `Extract Sora Job ID1`
    - Wait amount: `30` seconds

19. **Add `HTTP Request`**
    - Name: `Poll Sora Status1`
    - Connect from `Wait 30 Seconds1`
    - Method: `GET`
    - URL: `https://api.openai.com/v1/videos/{{ $json.videoId }}`
    - Authentication: OpenAI credential
    - Timeout: about 30000 ms

20. **Add `Code` node**
    - Name: `Check Sora Status1`
    - Connect from `Poll Sora Status1`
    - Merge status response with original product context
    - Increment `pollCount`
    - Throw if:
      - the response contains `error`
      - `pollCount >= 40`
    - Set `isDone = status === "completed"`

21. **Add `If` node**
    - Name: `Video Ready?1`
    - Connect from `Check Sora Status1`
    - Condition:
      - boolean equals
      - `{{$json.isDone}}`
      - `true`

22. **Connect the false branch**
    - Connect `Video Ready?1` false output back to `Wait 30 Seconds1`.

### Stage 6: Download video, upload to Drive, and update sheet

23. **Add `HTTP Request`**
    - Name: `Fetch Sora Video Content1`
    - Connect the true branch of `Video Ready?1`
    - Method: `GET`
    - URL: `https://api.openai.com/v1/videos/{{ $json.videoId }}/content`
    - Authentication: OpenAI credential
    - Response format: `File`
    - Timeout: about 120000 ms

24. **Add `Google Drive` node**
    - Name: `Upload Video to Google Drive1`
    - Connect from `Fetch Sora Video Content1`
    - Operation: upload file
    - Select drive: `My Drive`
    - Select folder: your target folder ID
    - File name expression:
      - `{{ $('Check Sora Status1').first().json.productName || 'ugc-video' }}_{{ $('Check Sora Status1').first().json.videoId }}.mp4`

25. **Add `HTTP Request`**
    - Name: `Make File Public1`
    - Connect from `Upload Video to Google Drive1`
    - Method: `POST`
    - URL:
      - `https://www.googleapis.com/drive/v3/files/{{ $json.id }}/permissions`
    - Authentication: predefined Google Drive OAuth2 credential
    - Send JSON body:
      - `role: "reader"`
      - `type: "anyone"`

26. **Add `Code` node**
    - Name: `Build Public Drive URL1`
    - Connect from `Make File Public1`
    - Build:
      - `videoUrl = https://drive.google.com/file/d/{fileId}/view?usp=sharing`
    - Restore/carry:
      - `productName`
      - `description`
      - `targetAudience`
      - `sheetRowNumber`
      - `imageUrl`
      - `videoId`

27. **Add `Google Sheets` node**
    - Name: `Update Sheet — Done2`
    - Connect from `Build Public Drive URL1`
    - Operation: `update`
    - Spreadsheet: same source document
    - Sheet: same products sheet used earlier
    - Match by column: `row_number`
    - Map fields:
      - `Status = Done`
      - `Image URL = {{$json.imageUrl}}`
      - `Video URL = {{$json.videoUrl}}`
      - `row_number = {{$json.sheetRowNumber}}`
      - `Description = {{$json.description}}`
      - `Product Name = {{$json.productName}}`
      - `Target Audience = {{$json.targetAudience}}`

28. **Close the batch loop**
    - Connect `Update Sheet — Done2` back to `Loop Over Items1`.

---

## C. Build the error workflow

29. **Create a second workflow**
    - This is separate from the main workflow.

30. **Add `Error Trigger`**
    - Name: `Error Trigger`

31. **Add `Code`**
    - Name: `Parse Error Details`
    - Connect from `Error Trigger`
    - Logic should:
      - inspect `execution.error.message`
      - inspect `execution.lastNodeExecuted`
      - traverse `execution.data.resultData.runData`
      - try to find the latest item containing `sheetRowNumber`
      - return:
        - `sheetRowNumber`
        - `productName`
        - `errorMessage`
        - `failedNode`
        - `executionId`
        - `timestamp`

32. **Add `If`**
    - Name: `Row Number Known?`
    - Connect from `Parse Error Details`
    - Condition: `sheetRowNumber exists`

33. **Add `Google Sheets`**
    - Name: `Update Sheet — Error`
    - Connect from true branch
    - Operation: `update`
    - Spreadsheet: same Google Sheet
    - Sheet: ensure it is the exact same tab as the main workflow
    - Match by `row_number`
    - Set:
      - `Status = Error`
      - `Error Message = {{$json.errorMessage + ' | Node: ' + $json.failedNode + ' | Execution: ' + $json.executionId}}`

34. **Add `Code`**
    - Name: `Log Error (Unknown Row)`
    - Connect from false branch
    - Log the error to console and return `logged: true`

35. **Assign the error workflow**
    - Open the main workflow settings.
    - Set this second workflow as the **Error Workflow**.

---

## D. Recommended corrections before production use

36. **Unify sheet naming**
   - The main workflow reads/writes `Sheet1` (`gid=0`), while the error workflow points to a sheet named `Products`.
   - Make both workflows use the same worksheet.

37. **Use a stable row identifier**
   - The current `sheetRowNumber = i + 2` may drift if reading behavior changes.
   - Prefer using an explicit ID column in the sheet, or the actual Google Sheets row metadata if available.

38. **Verify the `row_number` matching strategy**
   - The update nodes match by `row_number`.
   - Ensure this column exists in the sheet and contains the exact values the workflow expects.

39. **Validate OpenAI video access**
   - Confirm your OpenAI account has access to the `/v1/videos` endpoints and the specified video model.

40. **Test with one row first**
   - Add one `Pending` row and run the workflow manually before enabling the schedule.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This is an n8n automation workflow that turns product listings in a Google Sheet into UGC-style videos, fully hands-off. | General workflow description |
| It runs on a schedule, picks up any row marked "Pending" in the sheet, and for each product it: generates a product image with DALL·E 3, analyzes that image with GPT-4 Vision to inform the video style, sends a tailored prompt to Sora to generate a short video, uploads the finished video to Google Drive, and writes the public links back to the sheet. | General workflow description |
| The row's status gets updated to "Done" when complete, or "Error" with details if something fails along the way. | General workflow description |
| Before running this workflow, configure Google Sheets OAuth, Google Drive credentials, OpenAI API access, and sheet/folder references. | Setup context |
| The workflow expects a Google Sheet with product information, status tracking, media URL columns, and an error column. | Data model context |
| The error workflow must be linked in the main workflow settings under “Error Workflow”. | Operational setup |
| Recommended production improvement: replace computed row positions with a stable row ID column. | Reliability note |
| Recommended production improvement: make the main workflow sheet and error workflow sheet point to the exact same tab. | Consistency note |