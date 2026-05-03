Create a virtual outfit try-on Telegram bot with async polling and Google Sheets

https://n8nworkflows.xyz/workflows/create-a-virtual-outfit-try-on-telegram-bot-with-async-polling-and-google-sheets-14006


# Create a virtual outfit try-on Telegram bot with async polling and Google Sheets

# 1. Workflow Overview

This workflow implements a Telegram bot for virtual apparel try-on. A user first sends a **person photo**, then sends a **garment photo** with the caption `garment`. The workflow stores interim user state in Google Sheets, resolves Telegram file IDs into downloadable image URLs, submits both images to an external try-on API, polls asynchronously for job completion, and finally sends the rendered result back to the same Telegram chat.

The design solves a common Telegram automation constraint: each incoming message starts a separate workflow execution, so the workflow uses **Google Sheets as persistent state** keyed by `chat_id`.

## 1.1 Input Reception and Normalization
The workflow starts from a Telegram Trigger, extracts key message properties, and centralizes reusable configuration values such as Telegram token, Try-On API key/base URL, and Google Sheet ID.

## 1.2 Message Routing
The workflow checks whether the incoming Telegram message contains a photo. If not, it sends a welcome/instruction message. If a photo exists, it determines whether the image is a **person photo** or a **garment photo** based on the caption value `garment`.

## 1.3 Persistent State via Google Sheets
If the image is treated as a person photo, the workflow stores the Telegram `file_id` in Google Sheets. If it is treated as a garment photo, it looks up the previously saved person image for the same `chat_id`.

## 1.4 Try-On Job Preparation
When both image references are available, the workflow gathers all required IDs and configuration values, notifies the user that processing has started, and resolves Telegram `file_id` values into actual downloadable file URLs.

## 1.5 Try-On API Submission
The workflow downloads both images as binary files and submits them as multipart form-data to the external try-on API.

## 1.6 Asynchronous Polling Loop
Because the API is asynchronous, the workflow waits 15 seconds, checks the job status, and loops until the status becomes either `completed` or `failed`.

## 1.7 Result Delivery and Cleanup
On success, the generated image is downloaded and re-uploaded to Telegram as a file. On failure, an error message is sent. On success only, the saved Google Sheets state row is deleted to reset the interaction.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Configuration

### Overview
This block receives Telegram messages, extracts the fields needed later in the workflow, and builds a centralized configuration object. It ensures later nodes can reference a normalized set of variables instead of raw Telegram payload fields.

### Nodes Involved
- Telegram Trigger
- Extract Message Info
- ⚙️ Config

### Node Details

#### Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger`; entry point that listens for Telegram bot updates.
- **Configuration choices:** Configured to receive only `message` updates.
- **Key expressions / variables used:** None in node parameters; emits Telegram message payload.
- **Input / output connections:** Entry node; outputs to **Extract Message Info**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram credential missing or invalid
  - Workflow not activated; Telegram Trigger works only when active
  - Bot webhook or polling registration conflicts outside n8n
  - Non-message updates are ignored because only `message` is subscribed
- **Sub-workflow reference:** None

#### Extract Message Info
- **Type / role:** `Set`; normalizes incoming Telegram message data into simpler fields.
- **Configuration choices:**
  - `chatId` = Telegram chat ID
  - `caption` = lowercased and trimmed caption, defaulting to empty string
  - `hasPhoto` = boolean indicating whether `message.photo` exists
  - `fileId` = last photo variant’s `file_id` if present; otherwise empty string
- **Key expressions / variables used:**
  - `{{ $json.message.chat.id }}`
  - `{{ ($json.message.caption ?? '').toLowerCase().trim() }}`
  - `{{ $json.message.photo !== undefined }}`
  - `{{ $json.message.photo ? $json.message.photo[$json.message.photo.length - 1].file_id : '' }}`
- **Input / output connections:** Input from **Telegram Trigger**; output to **⚙️ Config**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Messages without `message.chat.id` would break, though Telegram message updates normally include it
  - Caption normalization assumes string-like caption or null
  - If Telegram photo array is unexpectedly empty, file selection could fail; current expression is safe only if `message.photo` exists and contains elements
- **Sub-workflow reference:** None

#### ⚙️ Config
- **Type / role:** `Set`; central configuration hub combining static setup values and normalized runtime values.
- **Configuration choices:**
  - Static placeholders:
    - `botToken` = `{TELEGRAM_BOT_TOKEN}`
    - `tryonApiKey` = `{TRYON_API_KEY}`
    - `tryonApiBase` = `https://tryon-api.com`
    - `sheetId` = `{GOOGLE_SHEET_ID}`
  - Runtime passthrough:
    - `chatId`, `fileId`, `caption`, `hasPhoto`
- **Key expressions / variables used:**
  - References to **Extract Message Info** via `$('Extract Message Info').item.json...`
- **Input / output connections:** Input from **Extract Message Info**; output to **Has Photo?**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Placeholder values must be replaced before production use
  - Incorrect `sheetId`, API key, or bot token will cause downstream failures
  - Heavy reliance on direct node references means renaming source nodes may break expressions if not updated automatically
- **Sub-workflow reference:** None

---

## Block 2 — Message Classification and Basic User Guidance

### Overview
This block decides whether the message contains a photo and, if so, whether that photo should be treated as a person image or a garment image. If the message is invalid for processing, the bot responds with instructions.

### Nodes Involved
- Has Photo?
- Send Welcome Message
- Is Garment Photo?

### Node Details

#### Has Photo?
- **Type / role:** `If`; branches based on whether the Telegram message contains a photo.
- **Configuration choices:** Checks `{{ $json.hasPhoto }}` is `true`.
- **Key expressions / variables used:**
  - `{{ $json.hasPhoto }}`
- **Input / output connections:** Input from **⚙️ Config**.
  - True branch → **Is Garment Photo?**
  - False branch → **Send Welcome Message**
- **Version-specific requirements:** Type version `2.2`, conditions version 2.
- **Edge cases / failures:**
  - If `hasPhoto` is absent due to upstream changes, strict validation may affect evaluation
  - Text-only messages always go to the welcome/instruction path
- **Sub-workflow reference:** None

#### Send Welcome Message
- **Type / role:** `Telegram`; sends usage instructions when the user has not sent a photo.
- **Configuration choices:**
  - Sends Markdown-formatted onboarding text
  - Uses chat ID from **⚙️ Config**
- **Key expressions / variables used:**
  - `{{ $('⚙️ Config').item.json.chatId }}`
- **Input / output connections:** Input from false branch of **Has Photo?**; no further output.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram auth error
  - Invalid chat ID if input structure changes
  - Markdown formatting could fail visually if edited incorrectly
- **Sub-workflow reference:** None

#### Is Garment Photo?
- **Type / role:** `If`; decides whether the current photo is a garment image.
- **Configuration choices:** Compares normalized caption to exact string `garment`.
- **Key expressions / variables used:**
  - `{{ $json.caption }}`
- **Input / output connections:** Input from true branch of **Has Photo?**
  - True branch → **Lookup Person from Sheet**
  - False branch → **Save Person to Sheet**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Caption must exactly equal `garment` after lowercasing and trimming
  - Any other caption, including descriptive text, causes the image to be treated as a person photo
  - A person photo accidentally sent with caption `garment` will be routed incorrectly
- **Sub-workflow reference:** None

---

## Block 3 — Persistent State Management in Google Sheets

### Overview
This block stores the user’s person photo after it is received, and later retrieves it when the garment photo arrives. It uses `chat_id` as the unique cross-execution key.

### Nodes Involved
- Save Person to Sheet
- Ask for Garment Photo
- Lookup Person from Sheet
- Has Person Saved?
- Ask for Person First

### Node Details

#### Save Person to Sheet
- **Type / role:** `Google Sheets`; persists person image state.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Sheet/tab: `tryon-state`
  - Matching column: `chat_id`
  - Writes:
    - `chat_id`
    - `person_file_id`
- **Key expressions / variables used:**
  - `{{ $('⚙️ Config').item.json.chatId }}`
  - `{{ $('⚙️ Config').item.json.fileId }}`
  - Document ID from config
- **Input / output connections:** Input from false branch of **Is Garment Photo?**; output to **Ask for Garment Photo**.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - OAuth2 credential invalid or expired
  - Sheet/tab missing
  - Header names not matching expected schema
  - Matching behavior depends on `chat_id` column existing exactly as configured
  - Concurrent updates from the same chat could overwrite previous person photo
- **Sub-workflow reference:** None

#### Ask for Garment Photo
- **Type / role:** `Telegram`; asks the user to send the garment image with the required caption.
- **Configuration choices:**
  - Markdown message
  - Uses chat ID from config
- **Key expressions / variables used:**
  - `{{ $('⚙️ Config').item.json.chatId }}`
- **Input / output connections:** Input from **Save Person to Sheet**; no further output.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram credential/chat delivery failure
- **Sub-workflow reference:** None

#### Lookup Person from Sheet
- **Type / role:** `Google Sheets`; retrieves previously stored person photo state for the current chat.
- **Configuration choices:**
  - Reads from sheet `tryon-state`
  - Filter: `chat_id = current chatId`
- **Key expressions / variables used:**
  - `{{ $json.chatId }}`
  - Document ID from **⚙️ Config**
- **Input / output connections:** Input from true branch of **Is Garment Photo?**; output to **Has Person Saved?**
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Sheet missing or unauthorized
  - No row found for chat
  - Multiple matching rows could produce ambiguous behavior depending on Google Sheets node output
  - If `chatId` is not present in current item, lookup fails logically
- **Sub-workflow reference:** None

#### Has Person Saved?
- **Type / role:** `If`; checks whether a stored person image exists.
- **Configuration choices:** Verifies `person_file_id` is not empty.
- **Key expressions / variables used:**
  - `{{ $json.person_file_id }}`
- **Input / output connections:** Input from **Lookup Person from Sheet**
  - True branch → **Collect IDs**
  - False branch → **Ask for Person First**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - If lookup returns no item rather than an item with empty field, behavior depends on node output semantics
  - Strict string comparison assumes the column exists
- **Sub-workflow reference:** None

#### Ask for Person First
- **Type / role:** `Telegram`; informs the user that a person photo must be sent before a garment photo.
- **Configuration choices:**
  - Markdown-formatted guidance
  - Uses chat ID from config
- **Key expressions / variables used:**
  - `{{ $('⚙️ Config').item.json.chatId }}`
- **Input / output connections:** Input from false branch of **Has Person Saved?**; no further output.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram delivery/auth issues
- **Sub-workflow reference:** None

---

## Block 4 — Job Context Preparation

### Overview
This block consolidates all data required for downstream processing and immediately informs the user that the try-on process has started. It also preserves values that would otherwise be hard to access later after branching and sheet lookup.

### Nodes Involved
- Collect IDs
- Send Processing Message

### Node Details

#### Collect IDs
- **Type / role:** `Set`; assembles all runtime identifiers and config values needed for API work and cleanup.
- **Configuration choices:** Produces:
  - `chatId`
  - `personFileId`
  - `garmentFileId`
  - `rowNumber`
  - `botToken`
  - `tryonApiKey`
  - `tryonApiBase`
  - `sheetId`
- **Key expressions / variables used:**
  - Current row values from Sheets: `person_file_id`, `row_number`
  - Config references from **⚙️ Config**
- **Input / output connections:** Input from true branch of **Has Person Saved?**; output to **Send Processing Message**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - `row_number` must exist if deletion is expected later
  - If Google Sheets output schema changes, fields may be missing
  - Placeholder config values propagate to downstream HTTP failures
- **Sub-workflow reference:** None

#### Send Processing Message
- **Type / role:** `Telegram`; tells the user the try-on generation is in progress.
- **Configuration choices:** Sends plain text to `chatId` from the current item.
- **Key expressions / variables used:**
  - `{{ $json.chatId }}`
- **Input / output connections:** Input from **Collect IDs**; output to **Get Person File Path**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram message could fail, but workflow will still not automatically branch around that unless execution stops
- **Sub-workflow reference:** None

---

## Block 5 — Telegram File Resolution and Binary Download

### Overview
Telegram image messages provide `file_id`, not direct file URLs. This block resolves both person and garment file IDs to file paths, constructs downloadable URLs, and downloads the two images as binary files.

### Nodes Involved
- Get Person File Path
- Set Person Download URL
- Get Garment File Path
- Set Garment Download URL
- Download Person Image
- Download Garment Image

### Node Details

#### Get Person File Path
- **Type / role:** `HTTP Request`; calls Telegram Bot API `getFile` for the person photo.
- **Configuration choices:**
  - GET request to `https://api.telegram.org/bot<TOKEN>/getFile`
  - Query parameter `file_id = personFileId`
- **Key expressions / variables used:**
  - URL built from `botToken`
  - Query from `personFileId`
- **Input / output connections:** Input from **Send Processing Message**; output to **Set Person Download URL**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Invalid bot token
  - Invalid or expired `file_id`
  - Telegram API rate limiting or transient HTTP errors
  - Unexpected response structure lacking `result.file_path`
- **Sub-workflow reference:** None

#### Set Person Download URL
- **Type / role:** `Set`; builds Telegram file download URL for the person image.
- **Configuration choices:** Constructs `personDownloadUrl` using bot token and Telegram `file_path`.
- **Key expressions / variables used:**
  - `https://api.telegram.org/file/bot{{ token }}/{{ $json.result.file_path }}`
- **Input / output connections:** Input from **Get Person File Path**; output to **Get Garment File Path**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing `result.file_path`
  - Invalid token produces unusable URL
- **Sub-workflow reference:** None

#### Get Garment File Path
- **Type / role:** `HTTP Request`; calls Telegram Bot API `getFile` for the garment photo.
- **Configuration choices:** Same pattern as person file lookup, using `garmentFileId`.
- **Key expressions / variables used:**
  - URL from `botToken`
  - Query from `garmentFileId`
- **Input / output connections:** Input from **Set Person Download URL**; output to **Set Garment Download URL**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Same failure types as **Get Person File Path**
- **Sub-workflow reference:** None

#### Set Garment Download URL
- **Type / role:** `Set`; builds Telegram file download URL for the garment image.
- **Configuration choices:** Creates `garmentDownloadUrl`.
- **Key expressions / variables used:**
  - `https://api.telegram.org/file/bot{{ token }}/{{ $json.result.file_path }}`
- **Input / output connections:** Input from **Get Garment File Path**; output to **Download Person Image**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing `result.file_path`
- **Sub-workflow reference:** None

#### Download Person Image
- **Type / role:** `HTTP Request`; downloads the person image as binary.
- **Configuration choices:**
  - URL from **Set Person Download URL**
  - Response format: file
  - Binary property name: `person_image`
- **Key expressions / variables used:**
  - `{{ $('Set Person Download URL').item.json.personDownloadUrl }}`
- **Input / output connections:** Input from **Set Garment Download URL**; output to **Download Garment Image**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Telegram file URL expired or inaccessible
  - Binary mode issues if workflow/environment is misconfigured
  - Large image downloads may impact memory or execution duration
- **Sub-workflow reference:** None

#### Download Garment Image
- **Type / role:** `HTTP Request`; downloads the garment image as binary.
- **Configuration choices:**
  - URL from **Set Garment Download URL**
  - Response format: file
  - Binary property name: `garment_image`
- **Key expressions / variables used:**
  - `{{ $('Set Garment Download URL').item.json.garmentDownloadUrl }}`
- **Input / output connections:** Input from **Download Person Image**; output to **Submit Try-On Job**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Same binary download concerns as the person image node
- **Sub-workflow reference:** None

---

## Block 6 — Try-On API Submission

### Overview
This block submits the person and garment images to the external try-on service and extracts the asynchronous job metadata required for polling.

### Nodes Involved
- Submit Try-On Job
- Extract Job ID

### Node Details

#### Submit Try-On Job
- **Type / role:** `HTTP Request`; submits multipart try-on request to external API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `{tryonApiBase}/api/v1/tryon`
  - Headers: `Authorization: Bearer <tryonApiKey>`
  - Content type: `multipart-form-data`
  - Body:
    - `person_images` from binary `person_image`
    - `garment_images` from binary `garment_image`
    - `fast_mode = false`
- **Key expressions / variables used:**
  - URL from `tryonApiBase`
  - Bearer token from `tryonApiKey`
- **Input / output connections:** Input from **Download Garment Image**; output to **Extract Job ID**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Invalid API key
  - Unsupported image formats or size limits
  - Multipart field names must match API expectations exactly
  - API may return non-200 response or a different response schema
  - Slow upstream service can cause timeouts
- **Sub-workflow reference:** None

#### Extract Job ID
- **Type / role:** `Set`; extracts `jobId` and `statusUrl` from API response.
- **Configuration choices:** Stores minimal polling state.
- **Key expressions / variables used:**
  - `{{ $json.jobId }}`
  - `{{ $json.statusUrl }}`
- **Input / output connections:** Input from **Submit Try-On Job**; output to **Wait 15 Seconds**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing `jobId` or `statusUrl` in response will break polling phase
- **Sub-workflow reference:** None

---

## Block 7 — Asynchronous Polling Loop

### Overview
This block repeatedly checks the status of the submitted try-on job until it either completes successfully or fails. It is implemented with a `Wait` node and conditional branching back into itself.

### Nodes Involved
- Wait 15 Seconds
- Check Job Status
- Merge State
- Is Job Complete?
- Is Job Failed?

### Node Details

#### Wait 15 Seconds
- **Type / role:** `Wait`; pauses execution between status checks.
- **Configuration choices:** Wait amount set to `15`.
- **Key expressions / variables used:** None
- **Input / output connections:**
  - Input from **Extract Job ID**
  - Also receives loopback from false branch of **Is Job Failed?**
  - Output to **Check Job Status**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Long-running executions may be affected by server retention or queue settings
  - Excessive loop duration could consume execution resources if API never resolves
- **Sub-workflow reference:** None

#### Check Job Status
- **Type / role:** `HTTP Request`; queries current job state.
- **Configuration choices:**
  - GET `{tryonApiBase}/api/v1/tryon/status/{jobId}`
  - Bearer Authorization header
- **Key expressions / variables used:**
  - Uses `jobId` from **Submit Try-On Job**
  - Uses `tryonApiBase` and `tryonApiKey` from **Collect IDs**
- **Input / output connections:** Input from **Wait 15 Seconds**; output to **Merge State**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Job not found
  - Invalid auth
  - Network timeout during polling
  - Response schema might not include `status`, `imageUrl`, or `error`
- **Sub-workflow reference:** None

#### Merge State
- **Type / role:** `Set`; normalizes the status response.
- **Configuration choices:** Stores:
  - `status`
  - `imageUrl`
  - `jobId`
  - `errorMessage`
- **Key expressions / variables used:**
  - `{{ $json.status }}`
  - `{{ $json.imageUrl }}`
  - `{{ $json.jobId }}`
  - `{{ $json.error ?? '' }}`
- **Input / output connections:** Input from **Check Job Status**; output to **Is Job Complete?**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing response fields may produce empty values
- **Sub-workflow reference:** None

#### Is Job Complete?
- **Type / role:** `If`; checks for terminal success status.
- **Configuration choices:** `status == completed`
- **Key expressions / variables used:**
  - `{{ $json.status }}`
- **Input / output connections:**
  - True branch → **HTTP Request** (download result image)
  - False branch → **Is Job Failed?**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Status values must exactly match expected string
  - Any non-`completed` status falls through, including `processing`, `queued`, or unexpected values
- **Sub-workflow reference:** None

#### Is Job Failed?
- **Type / role:** `If`; checks for terminal failure status.
- **Configuration choices:** `status == failed`
- **Key expressions / variables used:**
  - `{{ $json.status }}`
- **Input / output connections:**
  - True branch → **Send Error Message**
  - False branch → **Wait 15 Seconds** for continued polling
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Any status other than `failed` loops indefinitely, including unknown statuses
  - No max retry / timeout guard exists
- **Sub-workflow reference:** None

---

## Block 8 — Result Download, Delivery, and Cleanup

### Overview
This block handles terminal outcomes. On success, it downloads the generated result and uploads it to Telegram as a binary file, then removes the stored sheet state. On failure, it sends a user-facing error message.

### Nodes Involved
- HTTP Request
- Send Result Photo
- Delete Row from Sheet
- Send Error Message

### Node Details

#### HTTP Request
- **Type / role:** `HTTP Request`; downloads the completed try-on image from the returned `imageUrl`.
- **Configuration choices:** Uses direct URL from polling response.
- **Key expressions / variables used:**
  - `{{ $json.imageUrl }}`
- **Input / output connections:** Input from true branch of **Is Job Complete?**; output to **Send Result Photo**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - The result URL may expire
  - If the URL requires signed access and expires during delay, download fails
  - Response format is not explicitly configured as file in the JSON; sending as binary later assumes compatible node output behavior, so this should be validated in the target n8n version
- **Sub-workflow reference:** None

#### Send Result Photo
- **Type / role:** `Telegram`; sends the generated try-on image back to the user.
- **Configuration choices:**
  - Operation: `sendPhoto`
  - `binaryData = true`
  - Sends caption text
  - Uses `chatId` from **Collect IDs**
- **Key expressions / variables used:**
  - `{{ $('Collect IDs').item.json.chatId }}`
- **Input / output connections:** Input from **HTTP Request**; output to **Delete Row from Sheet**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Binary property mismatch if previous download node output is not in expected format
  - Telegram file upload size limits
  - Network/auth issues
- **Sub-workflow reference:** None

#### Delete Row from Sheet
- **Type / role:** `Google Sheets`; deletes stored person-photo state after successful completion.
- **Configuration choices:**
  - Operation: `delete`
  - Sheet: `tryon-state`
  - `startIndex = rowNumber`
- **Key expressions / variables used:**
  - `{{ $('Collect IDs').item.json.sheetId }}`
  - `{{ $('Collect IDs').item.json.rowNumber }}`
- **Input / output connections:** Input from **Send Result Photo**; terminal node.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - `rowNumber` indexing semantics must match the Google Sheets node expectation
  - If row numbering is off by one, the wrong row could be deleted
  - Deletion only occurs on success, so failures leave saved state in place
- **Sub-workflow reference:** None

#### Send Error Message
- **Type / role:** `Telegram`; informs the user that try-on processing failed.
- **Configuration choices:** Plain text error plus photo quality guidance.
- **Key expressions / variables used:**
  - `{{ $('Collect IDs').item.json.chatId }}`
- **Input / output connections:** Input from true branch of **Is Job Failed?**; terminal node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram delivery/auth issues
  - Since state is not deleted here, stale person-photo state persists intentionally or accidentally depending on desired design
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | Telegram Trigger | Receives Telegram message updates |  | Extract Message Info | # 👗 Virtual Try-On Telegram Bot  |
| Extract Message Info | Set | Normalizes Telegram payload into chat ID, caption, file ID, and photo flag | Telegram Trigger | ⚙️ Config | # 👗 Virtual Try-On Telegram Bot  |
| ⚙️ Config | Set | Centralizes tokens, sheet ID, API base URL, and normalized input fields | Extract Message Info | Has Photo? | ## ⚙️ Config — Edit This First |
| Has Photo? | If | Branches between instruction flow and photo-processing flow | ⚙️ Config | Is Garment Photo?, Send Welcome Message | ## ⚙️ Config — Edit This First |
| Send Welcome Message | Telegram | Sends onboarding instructions when no photo is provided | Has Photo? |  | ## ⚙️ Config — Edit This First |
| Is Garment Photo? | If | Distinguishes garment photo from person photo via caption `garment` | Has Photo? | Lookup Person from Sheet, Save Person to Sheet | ## 📋 State Management via Google Sheets |
| Save Person to Sheet | Google Sheets | Stores person photo file ID by chat ID | Is Garment Photo? | Ask for Garment Photo | ## 📋 State Management via Google Sheets |
| Ask for Garment Photo | Telegram | Prompts user to send garment photo | Save Person to Sheet |  | ## 📋 State Management via Google Sheets |
| Lookup Person from Sheet | Google Sheets | Retrieves saved person photo for the current chat | Is Garment Photo? | Has Person Saved? | ## 📋 State Management via Google Sheets |
| Has Person Saved? | If | Verifies whether a person photo exists in sheet state | Lookup Person from Sheet | Collect IDs, Ask for Person First | ## 📋 State Management via Google Sheets |
| Ask for Person First | Telegram | Informs user they must send person photo before garment photo | Has Person Saved? |  | ## 📋 State Management via Google Sheets |
| Collect IDs | Set | Carries all downstream runtime IDs and config fields | Has Person Saved? | Send Processing Message | ## 📁 Collect IDs |
| Send Processing Message | Telegram | Informs user that try-on is being processed | Collect IDs | Get Person File Path | ## 📁 Collect IDs |
| Get Person File Path | HTTP Request | Calls Telegram `getFile` for person photo | Send Processing Message | Set Person Download URL | ## 🔗 Telegram File URL Resolution |
| Set Person Download URL | Set | Builds direct Telegram file download URL for person image | Get Person File Path | Get Garment File Path | ## 🔗 Telegram File URL Resolution |
| Get Garment File Path | HTTP Request | Calls Telegram `getFile` for garment photo | Set Person Download URL | Set Garment Download URL | ## 🔗 Telegram File URL Resolution |
| Set Garment Download URL | Set | Builds direct Telegram file download URL for garment image | Get Garment File Path | Download Person Image | ## 🔗 Telegram File URL Resolution |
| Download Person Image | HTTP Request | Downloads person image as binary | Set Garment Download URL | Download Garment Image | ## 🔗 Telegram File URL Resolution |
| Download Garment Image | HTTP Request | Downloads garment image as binary | Download Person Image | Submit Try-On Job | ## 🔗 Telegram File URL Resolution |
| Submit Try-On Job | HTTP Request | Sends multipart try-on request to external API | Download Garment Image | Extract Job ID |  |
| Extract Job ID | Set | Stores job ID and status URL from API response | Submit Try-On Job | Wait 15 Seconds |  |
| Wait 15 Seconds | Wait | Delays before each polling attempt | Extract Job ID, Is Job Failed? | Check Job Status | ## ⏱️ Polling Loop |
| Check Job Status | HTTP Request | Queries try-on job status | Wait 15 Seconds | Merge State | ## ⏱️ Polling Loop |
| Merge State | Set | Normalizes polling response into status fields | Check Job Status | Is Job Complete? | ## ⏱️ Polling Loop |
| Is Job Complete? | If | Checks for completed status | Merge State | HTTP Request, Is Job Failed? | ## ⏱️ Polling Loop |
| Is Job Failed? | If | Checks for failed status or loops again | Is Job Complete? | Send Error Message, Wait 15 Seconds | ## ⏱️ Polling Loop |
| HTTP Request | HTTP Request | Downloads completed try-on result image | Is Job Complete? | Send Result Photo | ## ⬇️ Download Result Image |
| Send Result Photo | Telegram | Sends generated result image back to Telegram | HTTP Request | Delete Row from Sheet | ## ⬇️ Download Result Image |
| Delete Row from Sheet | Google Sheets | Removes saved person-photo state after success | Send Result Photo |  |  |
| Send Error Message | Telegram | Sends failure notification to user | Is Job Failed? |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Virtual Apparel Try-On Telegram Bot`.

2. **Add a Telegram Trigger node**.
   - Node type: **Telegram Trigger**
   - Configure it to listen for **message** updates only.
   - Assign your **Telegram bot credential**.
   - This node is the workflow entry point.

3. **Add a Set node named `Extract Message Info`** after the trigger.
   - Create these fields:
     1. `chatId` as string: `{{ $json.message.chat.id }}`
     2. `caption` as string: `{{ ($json.message.caption ?? '').toLowerCase().trim() }}`
     3. `hasPhoto` as boolean: `{{ $json.message.photo !== undefined }}`
     4. `fileId` as string: `{{ $json.message.photo ? $json.message.photo[$json.message.photo.length - 1].file_id : '' }}`
   - Connect **Telegram Trigger → Extract Message Info**.

4. **Add a Set node named `⚙️ Config`**.
   - Create these fields:
     1. `botToken` = your Telegram bot token
     2. `tryonApiKey` = your external try-on API key
     3. `tryonApiBase` = `https://tryon-api.com`
     4. `sheetId` = your Google Sheet ID
     5. `chatId` = `{{ $('Extract Message Info').item.json.chatId }}`
     6. `fileId` = `{{ $('Extract Message Info').item.json.fileId }}`
     7. `caption` = `{{ $('Extract Message Info').item.json.caption }}`
     8. `hasPhoto` = `{{ $('Extract Message Info').item.json.hasPhoto }}`
   - Replace the placeholder values with real values before enabling the workflow.
   - Connect **Extract Message Info → ⚙️ Config**.

5. **Add an If node named `Has Photo?`**.
   - Condition: boolean true
   - Left value: `{{ $json.hasPhoto }}`
   - True path = continue photo processing
   - False path = send instructions
   - Connect **⚙️ Config → Has Photo?**

6. **Add a Telegram node named `Send Welcome Message`** on the false branch.
   - Operation: send message
   - Chat ID: `{{ $('⚙️ Config').item.json.chatId }}`
   - Message:
     - Welcome text explaining:
       - send a person photo first
       - then send a garment photo with caption `garment`
   - Optional: set `parse_mode` to `Markdown`
   - Connect **Has Photo? false → Send Welcome Message**

7. **Add an If node named `Is Garment Photo?`** on the true branch.
   - Condition: string equals
   - Left value: `{{ $json.caption }}`
   - Right value: `garment`
   - True = garment branch
   - False = person branch
   - Connect **Has Photo? true → Is Garment Photo?**

8. **Create the Google Sheet used for state storage** before continuing.
   - In Google Sheets, create a spreadsheet.
   - Copy the spreadsheet ID from the URL.
   - Create a tab named exactly: `tryon-state`
   - Add at least these headers in row 1:
     - `chat_id`
     - `person_file_id`
   - Ensure your Google Sheets OAuth2 credential has access to this spreadsheet.

9. **Add a Google Sheets node named `Save Person to Sheet`** on the false branch of `Is Garment Photo?`.
   - Operation: `appendOrUpdate`
   - Document ID: `{{ $('⚙️ Config').item.json.sheetId }}`
   - Sheet name: `tryon-state`
   - Matching column: `chat_id`
   - Map columns:
     - `chat_id` = `{{ $('⚙️ Config').item.json.chatId }}`
     - `person_file_id` = `{{ $('⚙️ Config').item.json.fileId }}`
   - Assign your **Google Sheets OAuth2** credential.
   - Connect **Is Garment Photo? false → Save Person to Sheet**

10. **Add a Telegram node named `Ask for Garment Photo`**.
    - Operation: send message
    - Chat ID: `{{ $('⚙️ Config').item.json.chatId }}`
    - Message instructing user to send garment photo with caption `garment`
    - Optional Markdown parse mode
    - Connect **Save Person to Sheet → Ask for Garment Photo**

11. **Add a Google Sheets node named `Lookup Person from Sheet`** on the true branch of `Is Garment Photo?`.
    - Operation: read/search rows
    - Document ID: `{{ $('⚙️ Config').item.json.sheetId }}`
    - Sheet: `tryon-state`
    - Add filter:
      - lookup column: `chat_id`
      - lookup value: `{{ $json.chatId }}`
    - Connect **Is Garment Photo? true → Lookup Person from Sheet**

12. **Add an If node named `Has Person Saved?`**.
    - Condition: string not equals
    - Left value: `{{ $json.person_file_id }}`
    - Right value: empty string
    - True = continue
    - False = ask for person first
    - Connect **Lookup Person from Sheet → Has Person Saved?**

13. **Add a Telegram node named `Ask for Person First`** on the false branch.
    - Chat ID: `{{ $('⚙️ Config').item.json.chatId }}`
    - Message telling the user to send a person photo first, then the garment photo with caption `garment`
    - Optional Markdown parse mode
    - Connect **Has Person Saved? false → Ask for Person First**

14. **Add a Set node named `Collect IDs`** on the true branch.
    - Create fields:
      1. `chatId` = `{{ $('⚙️ Config').item.json.chatId }}`
      2. `personFileId` = `{{ $json.person_file_id }}`
      3. `garmentFileId` = `{{ $('⚙️ Config').item.json.fileId }}`
      4. `rowNumber` = `{{ $json.row_number }}`
      5. `botToken` = `{{ $('⚙️ Config').item.json.botToken }}`
      6. `tryonApiKey` = `{{ $('⚙️ Config').item.json.tryonApiKey }}`
      7. `tryonApiBase` = `{{ $('⚙️ Config').item.json.tryonApiBase }}`
      8. `sheetId` = `{{ $('⚙️ Config').item.json.sheetId }}`
    - Connect **Has Person Saved? true → Collect IDs**

15. **Add a Telegram node named `Send Processing Message`**.
    - Chat ID: `{{ $json.chatId }}`
    - Text indicating the request is processing and may take 15–60 seconds
    - Connect **Collect IDs → Send Processing Message**

16. **Add an HTTP Request node named `Get Person File Path`**.
    - Method: GET
    - URL: `https://api.telegram.org/bot{{ $('Collect IDs').item.json.botToken }}/getFile`
    - Enable query parameters
    - Add query parameter:
      - `file_id` = `{{ $('Collect IDs').item.json.personFileId }}`
    - Connect **Send Processing Message → Get Person File Path**

17. **Add a Set node named `Set Person Download URL`**.
    - Field:
      - `personDownloadUrl` = `https://api.telegram.org/file/bot{{ $('Collect IDs').item.json.botToken }}/{{ $json.result.file_path }}`
    - Connect **Get Person File Path → Set Person Download URL**

18. **Add an HTTP Request node named `Get Garment File Path`**.
    - Method: GET
    - URL: `https://api.telegram.org/bot{{ $('Collect IDs').item.json.botToken }}/getFile`
    - Query parameter:
      - `file_id` = `{{ $('Collect IDs').item.json.garmentFileId }}`
    - Connect **Set Person Download URL → Get Garment File Path**

19. **Add a Set node named `Set Garment Download URL`**.
    - Field:
      - `garmentDownloadUrl` = `https://api.telegram.org/file/bot{{ $('Collect IDs').item.json.botToken }}/{{ $json.result.file_path }}`
    - Connect **Get Garment File Path → Set Garment Download URL**

20. **Add an HTTP Request node named `Download Person Image`**.
    - URL: `{{ $('Set Person Download URL').item.json.personDownloadUrl }}`
    - Response format: **File**
    - Output binary property: `person_image`
    - Connect **Set Garment Download URL → Download Person Image**

21. **Add an HTTP Request node named `Download Garment Image`**.
    - URL: `{{ $('Set Garment Download URL').item.json.garmentDownloadUrl }}`
    - Response format: **File**
    - Output binary property: `garment_image`
    - Connect **Download Person Image → Download Garment Image**

22. **Add an HTTP Request node named `Submit Try-On Job`**.
    - Method: `POST`
    - URL: `{{ $('Collect IDs').item.json.tryonApiBase }}/api/v1/tryon`
    - Send headers: yes
    - Header:
      - `Authorization` = `Bearer {{ $('Collect IDs').item.json.tryonApiKey }}`
    - Body content type: `multipart-form-data`
    - Body fields:
      - `person_images` as binary from `person_image`
      - `garment_images` as binary from `garment_image`
      - `fast_mode` = `false`
    - Connect **Download Garment Image → Submit Try-On Job**

23. **Add a Set node named `Extract Job ID`**.
    - Fields:
      - `jobId` = `{{ $json.jobId }}`
      - `statusUrl` = `{{ $json.statusUrl }}`
    - Connect **Submit Try-On Job → Extract Job ID**

24. **Add a Wait node named `Wait 15 Seconds`**.
    - Wait amount: `15` seconds
    - Connect **Extract Job ID → Wait 15 Seconds**

25. **Add an HTTP Request node named `Check Job Status`**.
    - Method: GET
    - URL: `{{ $('Collect IDs').item.json.tryonApiBase }}/api/v1/tryon/status/{{ $('Submit Try-On Job').item.json.jobId }}`
    - Header:
      - `Authorization` = `Bearer {{ $('Collect IDs').item.json.tryonApiKey }}`
    - Connect **Wait 15 Seconds → Check Job Status**

26. **Add a Set node named `Merge State`**.
    - Fields:
      - `status` = `{{ $json.status }}`
      - `imageUrl` = `{{ $json.imageUrl }}`
      - `jobId` = `{{ $json.jobId }}`
      - `errorMessage` = `{{ $json.error ?? '' }}`
    - Connect **Check Job Status → Merge State**

27. **Add an If node named `Is Job Complete?`**.
    - Condition:
      - `{{ $json.status }}` equals `completed`
    - True = success path
    - False = check failure path
    - Connect **Merge State → Is Job Complete?**

28. **Add an If node named `Is Job Failed?`**.
    - Condition:
      - `{{ $json.status }}` equals `failed`
    - True = send error
    - False = loop again
    - Connect **Is Job Complete? false → Is Job Failed?**

29. **Create the polling loop**.
    - Connect **Is Job Failed? false → Wait 15 Seconds**
    - This causes the workflow to re-check status until it becomes `completed` or `failed`.

30. **Add an HTTP Request node named `HTTP Request`** on the success path.
    - URL: `{{ $json.imageUrl }}`
    - Prefer configuring the response as a **file/binary output** for reliability
    - Connect **Is Job Complete? true → HTTP Request**

31. **Add a Telegram node named `Send Result Photo`**.
    - Operation: `sendPhoto`
    - `binaryData` = true
    - Chat ID: `{{ $('Collect IDs').item.json.chatId }}`
    - Caption: success message
    - Ensure the incoming binary property matches what this node expects
    - Connect **HTTP Request → Send Result Photo**

32. **Add a Google Sheets node named `Delete Row from Sheet`**.
    - Operation: `delete`
    - Document ID: `{{ $('Collect IDs').item.json.sheetId }}`
    - Sheet name: `tryon-state`
    - Start index: `{{ $('Collect IDs').item.json.rowNumber }}`
    - Connect **Send Result Photo → Delete Row from Sheet**

33. **Add a Telegram node named `Send Error Message`** on the failed path.
    - Chat ID: `{{ $('Collect IDs').item.json.chatId }}`
    - Text explaining the try-on failed and suggesting clearer photos
    - Connect **Is Job Failed? true → Send Error Message**

34. **Configure credentials**.
    - Telegram:
      - Use one Telegram credential for the trigger and all Telegram send nodes
    - Google Sheets:
      - Use one Google Sheets OAuth2 credential for all Sheets nodes
    - External Try-On API:
      - No dedicated credential object is used in this workflow; API key is stored in the `⚙️ Config` node and passed as a bearer token header

35. **Check workflow settings**.
    - Binary mode in the provided workflow is `separate`
    - Execution order is `v1`
    - If reproducing in another environment, ensure binary handling is compatible with file download/upload nodes

36. **Test the workflow manually**.
    - Send a text message to the bot: should return welcome instructions
    - Send a person photo without caption: should save state and ask for garment photo
    - Send a garment photo with caption `garment`: should start processing and later return a result or error

37. **Activate the workflow**.
    - Telegram Trigger only functions properly when the workflow is active.

### Important implementation constraints
1. The Google Sheet must contain the `tryon-state` tab and proper headers.
2. The `caption` must normalize exactly to `garment` for the garment route.
3. The result-download node should ideally be explicitly configured for binary/file output to avoid upload issues.
4. There is no max retry count in the polling loop; consider adding one if you want bounded execution time.
5. On failure, the row is not deleted; decide whether you want to preserve or clear state in that case.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and is not itself documented as a sub-workflow. There are no `Execute Workflow` nodes or external n8n workflow dependencies.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This bot lets users virtually try on clothing via Telegram. A user sends a person photo, then a garment photo with caption `garment`, and the bot replies with an AI-generated try-on result image. | Workflow purpose |
| Setup checklist: set Telegram Bot Token in `⚙️ Config`; set Google Sheet ID in `⚙️ Config`; confirm Google Sheet has tab `tryon-state` with headers `chat_id` and `person_file_id`; assign Telegram credentials to all Telegram nodes; assign Google Sheets credentials to all Google Sheets nodes; activate the workflow. | Operational setup |
| Google Sheets is used as persistent state because each Telegram message creates a separate workflow execution with no shared memory. | Architecture note |
| The `chat_id` acts as the unique key linking the saved person photo to the later garment photo. | State management logic |
| Telegram file messages do not provide direct download URLs; `getFile` must be called first to retrieve `file_path`, then a file download URL must be constructed. | Telegram API behavior |
| The Try-On API is asynchronous and returns a `jobId`, so polling every 15 seconds is used until the job is `completed` or `failed`. | External API behavior |
| Downloading the result image before sending it to Telegram is more reliable than passing a signed URL directly, especially if the URL expires or requires special access conditions. | Result delivery note |