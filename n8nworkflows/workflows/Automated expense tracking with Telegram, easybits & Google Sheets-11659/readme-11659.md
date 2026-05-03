Automated expense tracking with Telegram, easybits & Google Sheets

https://n8nworkflows.xyz/workflows/automated-expense-tracking-with-telegram--easybits---google-sheets-11659


# Automated expense tracking with Telegram, easybits & Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Telegram Expense Bot  
**Purpose:** Receive a receipt (photo or PDF) via Telegram, extract structured expense data using easybits (OCR/data extraction), then log it into a **monthly Google Sheets tab** (e.g., `December_2025`) with formatting, and finally send a confirmation back to the user in Telegram.

**Target use cases**
- Freelancers/small businesses logging expenses quickly from a phone
- Semi-automated bookkeeping with consistent monthly sheets and totals
- Basic reimbursement handling (e.g., 80% coverage for “Mobile Phone”)

**Logical blocks (based on node dependencies)**
1.1 **Input reception & validation (Telegram)**  
1.2 **Download file from Telegram & normalize payload**  
1.3 **Extraction via easybits & normalization/business rules**  
1.4 **Unknown category handling (human-in-the-loop via caption)**  
1.5 **Google Sheet preparation (create monthly sheet + headers if missing)**  
1.6 **Append expense row & format the row**  
1.7 **Telegram confirmation & webhook response**

---

## 2. Block-by-Block Analysis

### 1.1 Input reception & validation (Telegram)

**Overview:** Receives incoming Telegram updates through a webhook endpoint and verifies that the message includes either a receipt photo or a PDF document. If not, it warns the user.

**Nodes involved**
- Telegram Webhook
- Has Photo or PDF?
- No Photo Message
- Respond OK 2

#### Node: **Telegram Webhook**
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) – entry point to receive Telegram update payloads.
- **Configuration (interpreted):**
  - Method: `POST`
  - Path: `expense-bot` (webhookId also “expense-bot”)
  - Response mode: `responseNode` (workflow must end in Respond to Webhook)
- **Inputs/Outputs:**
  - Output → **Has Photo or PDF?**
- **Edge cases / failures:**
  - Telegram webhook not set correctly at Telegram side (setWebhook to this URL).
  - Telegram may resend updates if response is slow; the workflow responds “OK” later, so latency matters.

#### Node: **Has Photo or PDF?**
- **Type / role:** `IF` – checks presence of receipt media.
- **Key logic:**
  - Evaluates: `{{ $json.body.message.photo || $json.body.message.document ? 'yes' : 'no' }} == 'yes'`
- **Outputs:**
  - **True** → Get File Info
  - **False** → No Photo Message
- **Edge cases:**
  - Telegram messages that contain neither `photo` nor `document` (text-only, stickers, etc.) go to error branch.
  - If Telegram sends `channel_post` or other update types, `body.message` may be missing → expression failure risk (strict validation).

#### Node: **No Photo Message**
- **Type / role:** `HTTP Request` – sends message back via Telegram Bot API.
- **Configuration:**
  - POST `https://api.telegram.org/bot<TOKEN>/sendMessage`
  - Hardcoded `chat_id: 413057682`
  - Text: “Please send a photo…”
- **Outputs:**
  - → Respond OK 2
- **Edge cases / failures:**
  - **Hardcoded chat_id** means it will always message a single user, not the sender.
  - Bot token exposed in URL (security risk).
  - If bot blocked by user, Telegram API returns error.

#### Node: **Respond OK 2**
- **Type / role:** `Respond to Webhook` – ends webhook execution for the “no media” path.
- **Configuration:** Responds with text `"OK"`.
- **Outputs:** none

---

### 1.2 Download file from Telegram & normalize payload

**Overview:** Uses Telegram Bot API to resolve `file_id` → `file_path`, downloads the actual file, and converts it to a Base64 data URI while also capturing chat id and mime type.

**Nodes involved**
- Get File Info
- Download Photo
- Convert to Base64

#### Node: **Get File Info**
- **Type / role:** `HTTP Request` – Telegram `getFile` call.
- **Configuration:**
  - POST `https://api.telegram.org/bot<TOKEN>/getFile`
  - JSON body picks `file_id` from:
    - last photo size: `$json.body.message.photo.slice(-1)[0].file_id`
    - OR document: `$json.body.message.document.file_id`
- **Inputs/Outputs:**
  - Input: from Has Photo or PDF?
  - Output → Download Photo
- **Edge cases / failures:**
  - If message has `photo` but array empty/unexpected: `slice(-1)[0]` may fail.
  - Bot token in URL (security).
  - Telegram API rate limits.

#### Node: **Download Photo**
- **Type / role:** `HTTP Request` – downloads the file bytes from Telegram file endpoint.
- **Configuration:**
  - GET `https://api.telegram.org/file/bot<TOKEN>/<file_path>`
  - Response format: `file` (binary)
- **Outputs:** → Convert to Base64
- **Edge cases / failures:**
  - Very large PDFs can hit n8n memory limits depending on hosting.
  - Telegram file_path can expire/require re-fetch if delayed (rare).

#### Node: **Convert to Base64**
- **Type / role:** `Code` – normalizes binary data to data URI and enriches payload.
- **Key logic / variables:**
  - Detects first binary key dynamically: `Object.keys(inputData.binary || {})[0]`
  - `base64 = binaryData.data` (n8n stores binary data as base64 string internally)
  - Determines mime type from original Telegram message:
    - document: `telegramMessage.document.mime_type` default `application/pdf`
    - photo: `image/jpeg`
  - Gets `chatId` from webhook: `$('Telegram Webhook').first().json.body.message.chat.id`
  - Outputs:
    - `dataUri`, `chatId`, `mimeType`, debug fields
- **Outputs:** → Extract with easybits
- **Edge cases / failures:**
  - If binary is not present (download failed), `binaryData` undefined → runtime error.
  - Using `$()` references assumes upstream nodes executed and data exists.
  - Data URI is produced but the next node actually sends a URL to easybits, not the data URI (so this node is currently used mostly for `chatId` + mime inference/debug).

---

### 1.3 Extraction via easybits & normalization/business rules

**Overview:** Sends the Telegram-hosted file URL to an easybits extraction pipeline, then transforms the extraction output into a consistent expense structure (category/vendor/date/amount), determines the monthly sheet name, and applies reimbursement logic.

**Nodes involved**
- Extract with easybits
- Process Extraction

#### Node: **Extract with easybits**
- **Type / role:** `HTTP Request` – calls easybits extractor pipeline endpoint.
- **Configuration:**
  - POST `https://extractor.easybits.tech/api/pipelines/dYI3r9xfHldQBlei34Pe`
  - Body: `{ files: [ "https://api.telegram.org/file/bot<TOKEN>/" + <file_path> ] }`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder)
    - `Content-Type: application/json`
- **Inputs/Outputs:**
  - Depends on `Get File Info` output via expression for `file_path`
  - Output → Process Extraction
- **Edge cases / failures:**
  - If easybits requires fetching the URL, Telegram file URL must be publicly reachable; it is, but requires bot token embedded in URL (security).
  - Unauthorized/expired token → 401.
  - Pipeline schema changes → parsing assumptions in Process Extraction break.

#### Node: **Process Extraction**
- **Type / role:** `Code` – maps extractor output → expense fields; caption-based overrides; month/year routing.
- **Key logic:**
  - Reads extraction result and tries multiple shapes:
    - `extractionResult.data.receipt_data`
    - `extractionResult.receipt_data`
    - fallback `extractionResult.data || extractionResult`
  - Defaults:
    - `category = 'Other'` if missing
    - `vendor_name = 'Unknown'`
    - `total_amount = parseFloat(...) || 0`
    - `currency = 'EUR'`
    - `transaction_date` fallback: current date in `de-DE` locale
  - Caption parsing (from Telegram message caption):
    - Supports `Category - details`
    - Only accepts categories in `validCategories` list (Restaurant, Transportation, Mobile Phone, Office Supplies, Accommodation, Marketing, Software, Equipment)
    - If caption doesn’t match a category, treats it as details
  - Reimbursement logic:
    - `eur_rate = 0.8` when category is `Mobile Phone`, else `1`
    - `eur_amount = total_amount * eur_rate`
  - Month/year/sheet name:
    - Tries parsing `transaction_date` as `dd.mm.yyyy`
    - `sheet_name = <MonthName>_<Year>`
    - Defaults month/year to `December`/`2025` if parsing fails
  - Details logic: derives details text depending on category/caption
  - Flag:
    - `needs_category_input = (category === 'Other')`
- **Outputs:** → Needs Category?
- **Edge cases / failures:**
  - Date parsing assumes dot-separated German date; ISO (`2025-12-01`) will fall back to December_2025 regardless of actual month.
  - `parseFloat` on amounts like `12,34` becomes `12` (comma decimal issues).
  - Hardcoded reimbursement rule only for one category.
  - If extractor returns strings with currency symbols, amount parsing may degrade.

---

### 1.4 Unknown category handling (human-in-the-loop)

**Overview:** If the extracted/derived category is “Other”, the workflow asks the user to resend the receipt with a caption specifying category and optional details; otherwise it continues to Google Sheets logging.

**Nodes involved**
- Needs Category?
- Ask for Category
- Respond OK

#### Node: **Needs Category?**
- **Type / role:** `IF` – routes based on `needs_category_input`.
- **Condition:** `{{ $json.needs_category_input }}` is true.
- **Outputs:**
  - **True** → Ask for Category
  - **False** → Check Sheet Exists
- **Edge cases:**
  - If `needs_category_input` missing/null, “loose” validation may behave unexpectedly.

#### Node: **Ask for Category**
- **Type / role:** `Telegram` node – sends message via Telegram API using n8n Telegram credentials (safer than raw bot token).
- **Configuration:**
  - Chat ID: `{{ $('Process Extraction').item.json.chat_id }}`
  - Text instructs: resend with caption `Category - description` and lists allowed categories.
- **Outputs:** → Respond OK
- **Edge cases / failures:**
  - If Telegram credential not configured, node fails.
  - This does not “pause” the workflow; it just asks and ends. The user must resend and trigger workflow again.

#### Node: **Respond OK**
- **Type / role:** `Respond to Webhook` – ends webhook execution for both “asked category” and “success” paths.
- **Configuration:** Responds `"OK"`.
- **Edge cases:**
  - Node is shared by both Ask for Category and Send Confirmation; if one branch errors before reaching it, Telegram may retry webhook.

---

### 1.5 Google Sheet preparation (create monthly sheet + headers if missing)

**Overview:** Checks if the monthly sheet tab exists. If not, creates it and applies a full header + formatting layout via `batchUpdate`. If it exists, it retrieves its `sheetId` for later formatting operations.

**Nodes involved**
- Check Sheet Exists
- Sheet Exists?
- Create Sheet
- Build Headers Request
- Setup Headers
- Get Sheet Info
- Extract Sheet ID

#### Node: **Check Sheet Exists**
- **Type / role:** `HTTP Request` – attempts to read cell A1 in the monthly tab.
- **Configuration:**
  - GET `.../values/'<sheet_name>'!A1`
  - `neverError: true` so 404/400 won’t fail node
  - Auth: Google Sheets OAuth2 credential
- **Outputs:** → Sheet Exists?
- **Edge cases / failures:**
  - If sheet name has characters requiring encoding, quoting may be insufficient.
  - API may return errors that still don’t set `values` (used downstream).

#### Node: **Sheet Exists?**
- **Type / role:** `IF` – checks whether response includes `values`.
- **Condition:** `{{ $json.values ? 'yes' : 'no' }} == 'no'`
  - If “no values”, assumes sheet does not exist.
- **Outputs:**
  - **True (sheet missing)** → Create Sheet
  - **False (exists)** → Get Sheet Info
- **Edge cases:**
  - A sheet can exist but A1 be empty; then `values` may be absent, causing false “missing sheet” and failing create due to duplicate title.

#### Node: **Create Sheet**
- **Type / role:** `HTTP Request` – `spreadsheets:batchUpdate` with `addSheet`.
- **Configuration:**
  - POST should be: `https://sheets.googleapis.com/v4/spreadsheets/<spreadsheetId>:batchUpdate`
  - **In workflow JSON it is malformed:** `1EE-YOUR_AWS_SECRET_KEY_HERE:batchUpdate` (looks like a placeholder accidentally inserted).
  - On error: `continueRegularOutput` enabled (so it won’t stop even if create fails)
- **Outputs:** → Build Headers Request
- **Edge cases / failures:**
  - Malformed URL prevents sheet creation.
  - If sheet already exists, addSheet returns error; workflow continues but downstream assumes create response contains `sheetId`.

#### Node: **Build Headers Request**
- **Type / role:** `Code` – builds a Google Sheets `batchUpdate` request to format header rows, pre-format data rows, set column widths, and create a TOTAL row formula.
- **Key dependencies:**
  - Reads `monthName` / `year` from Process Extraction
  - Reads `sheetId` from Create Sheet response: `replies[0].addSheet.properties.sheetId`
- **Outputs:** → Setup Headers
- **Edge cases / failures:**
  - If Create Sheet failed or URL wrong, `sheetId` becomes `0` → formatting may target wrong sheet (sheetId 0 is often the first sheet, not the new one).

#### Node: **Setup Headers**
- **Type / role:** `HTTP Request` – posts the `batchUpdate` formatting request to Sheets API.
- **Configuration:**
  - POST `.../<spreadsheetId>:batchUpdate`
  - Body: `{{ $json.requestBody }}`
- **Outputs:** → Build Expense Data
- **Edge cases:**
  - If `requestBody` missing, request fails.
  - Requires Sheets scope allowing batchUpdate.

#### Node: **Get Sheet Info**
- **Type / role:** `HTTP Request` – fetches sheet list and properties to locate a tab by title.
- **Configuration:**
  - GET should be: `.../v4/spreadsheets/<spreadsheetId>?fields=sheets(properties(sheetId,title))`
  - **In workflow JSON it is malformed:** includes `1EE-YOUR_AWS_SECRET_KEY_HERE` placeholder.
- **Outputs:** → Extract Sheet ID
- **Edge cases:**
  - Malformed URL prevents sheetId retrieval, breaking later formatting.

#### Node: **Extract Sheet ID**
- **Type / role:** `Code` – finds `sheetId` matching `sheet_name`.
- **Logic:** loops through `sheetsData.sheets[]` and matches `sheet.properties.title === sheetName`.
- **Outputs:** → Build Expense Data
- **Edge cases:**
  - If Get Sheet Info failed or returned unexpected structure, `sheetId` remains `0`.

---

### 1.6 Append expense row & format the row

**Overview:** Builds the row payload, appends it to the monthly sheet starting at A4, then applies bold formatting to the newly added row using the returned updated range to compute the row index.

**Nodes involved**
- Build Expense Data
- Add Expense Row
- Build Row Format
- Format Data Row

#### Node: **Build Expense Data**
- **Type / role:** `Code` – constructs the `values` payload for append and carries sheetId for formatting.
- **Key logic:**
  - Coerces `total_amount` and `eur_amount` into numbers; strips non-numeric chars if needed.
  - Attempts to get `sheetId` from either:
    - Build Headers Request (new sheet path)
    - Extract Sheet ID (existing sheet path)
  - Outputs `expenseBody.values = [[date, category, vendor, details, amount, currency, eurAmount]]`
- **Outputs:** → Add Expense Row
- **Edge cases:**
  - The `sheetId` resolution code has a logical quirk: it uses `if (!sheetId && sheetId !== 0)` which is always false for `sheetId=0`; then it later special-cases `sheetId===0`. Net effect: it often ends with 0 unless Extract Sheet ID succeeded.
  - Amount parsing still may mis-handle comma decimals.

#### Node: **Add Expense Row**
- **Type / role:** `HTTP Request` – appends to sheet.
- **Configuration:**
  - POST `.../values/<sheet_name>!A4:G4:append?valueInputOption=USER_ENTERED&insertDataOption=OVERWRITE`
  - Body: `{{ $json.expenseBody }}`
- **Outputs:** → Build Row Format
- **Edge cases:**
  - `insertDataOption=OVERWRITE` with append is unusual; typically append inserts. Depending on Sheets behavior, might overwrite.
  - If the sheet doesn’t exist, API returns error.
  - If sheet_name includes spaces, proper quoting/encoding needed.

#### Node: **Build Row Format**
- **Type / role:** `Code` – builds a `batchUpdate` request to format exactly the appended row.
- **Key logic:**
  - Reads `updatedRange` from Add Expense Row response: `updates.updatedRange`
  - Extracts row number from pattern `!A(\d+)`
  - Uses `sheetId` from Build Expense Data
  - Applies:
    - white background
    - black bold text
    - wrap
- **Outputs:** → Format Data Row
- **Edge cases:**
  - If updatedRange missing, defaults to row 4.
  - If sheetId is 0 (wrong), format may apply to wrong tab.

#### Node: **Format Data Row**
- **Type / role:** `HTTP Request` – sends `batchUpdate` formatting request.
- **Configuration:**
  - POST should be: `.../v4/spreadsheets/<spreadsheetId>:batchUpdate`
  - **In workflow JSON it is malformed:** `1EE-YOUR_AWS_SECRET_KEY_HERE:batchUpdate`
  - Adds header `Content-Type:  application/json` (note leading space)
  - Auth: Google Sheets OAuth2
- **Outputs:** → Send Confirmation
- **Edge cases:**
  - Malformed URL breaks formatting.
  - Leading space in Content-Type value could cause issues in strict servers (Google usually ignores).
  - If sheetId invalid, API errors.

---

### 1.7 Telegram confirmation & webhook response

**Overview:** Sends a success message to the user with extracted details (category/vendor/amount/date/coverage/EUR value/sheet name), then returns “OK” to Telegram webhook.

**Nodes involved**
- Send Confirmation
- Respond OK

#### Node: **Send Confirmation**
- **Type / role:** `HTTP Request` – Telegram `sendMessage`.
- **Configuration:**
  - POST `https://api.telegram.org/bot<TOKEN>/sendMessage`
  - Uses chat_id from Process Extraction
  - Message includes:
    - coverage computed: `eur_rate === 1 ? '100%' : '80%'`
    - EUR amount with `.toFixed(2)`
- **Outputs:** → Respond OK
- **Edge cases:**
  - `.toFixed(2)` assumes eur_amount is a number; it is computed earlier but can still be non-number if extraction yielded NaN and logic changed.
  - Bot token exposure.

#### Node: **Respond OK**
- Covered earlier; used also as terminal node after Ask for Category and after Send Confirmation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Webhook | Webhook | Receive Telegram update payload | — | Has Photo or PDF? | ## 1. Receive input<br>Receive receipt from Telegram. Check if message contains a photo or PDF. |
| Has Photo or PDF? | IF | Validate media presence (photo/document) | Telegram Webhook | Get File Info; No Photo Message | ## 1. Receive input<br>Receive receipt from Telegram. Check if message contains a photo or PDF. |
| No Photo Message | HTTP Request | Notify user (but hardcoded chat_id) | Has Photo or PDF? | Respond OK 2 | ## 1. Receive input<br>Receive receipt from Telegram. Check if message contains a photo or PDF. |
| Respond OK 2 | Respond to Webhook | End webhook on “no photo” path | No Photo Message | — | ## 1. Receive input<br>Receive receipt from Telegram. Check if message contains a photo or PDF. |
| Get File Info | HTTP Request | Telegram getFile (file_id → file_path) | Has Photo or PDF? | Download Photo | ## 2. Download file<br>Download file from Telegram servers and convert to Base64 for easybits API. |
| Download Photo | HTTP Request | Download binary file from Telegram | Get File Info | Convert to Base64 | ## 2. Download file<br>Download file from Telegram servers and convert to Base64 for easybits API. |
| Convert to Base64 | Code | Build data URI + chatId + mimeType | Download Photo | Extract with easybits | ## 2. Download file<br>Download file from Telegram servers and convert to Base64 for easybits API. |
| Extract with easybits | HTTP Request | Call easybits pipeline for receipt extraction | Convert to Base64 | Process Extraction | ## 3. Extract data<br>Send file to easybits for extraction. Parse vendor, amount, date, and category. |
| Process Extraction | Code | Normalize extracted data + caption overrides + sheet routing | Extract with easybits | Needs Category? | ## 3. Extract data<br>Send file to easybits for extraction. Parse vendor, amount, date, and category. |
| Needs Category? | IF | Branch if category is “Other” | Process Extraction | Ask for Category; Check Sheet Exists | ## 4. Handle unknown category<br>If category is "Other", ask user to resend with a caption specifying the category. |
| Ask for Category | Telegram | Ask user to resend with category caption | Needs Category? | Respond OK | ## 4. Handle unknown category<br>If category is "Other", ask user to resend with a caption specifying the category. |
| Check Sheet Exists | HTTP Request | Probe A1 to infer if monthly sheet tab exists | Needs Category? | Sheet Exists? | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Sheet Exists? | IF | Decide create-vs-existing sheet path | Check Sheet Exists | Create Sheet; Get Sheet Info | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Create Sheet | HTTP Request | Add sheet tab (batchUpdate) | Sheet Exists? | Build Headers Request | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Build Headers Request | Code | Build batchUpdate request (headers, formatting, totals) | Create Sheet | Setup Headers | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Setup Headers | HTTP Request | Apply batchUpdate formatting to the new sheet | Build Headers Request | Build Expense Data | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Get Sheet Info | HTTP Request | List sheets + titles to locate sheetId | Sheet Exists? | Extract Sheet ID | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Extract Sheet ID | Code | Extract sheetId for an existing monthly sheet tab | Get Sheet Info | Build Expense Data | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Build Expense Data | Code | Build row values payload; propagate sheetId | Setup Headers / Extract Sheet ID | Add Expense Row | ## 6. Add expense row<br>Add expense to the correct monthly sheet. Apply bold formatting to the data row. |
| Add Expense Row | HTTP Request | Append expense values to monthly sheet | Build Expense Data | Build Row Format | ## 6. Add expense row<br>Add expense to the correct monthly sheet. Apply bold formatting to the data row. |
| Build Row Format | Code | Build batchUpdate request to format appended row | Add Expense Row | Format Data Row | ## 6. Add expense row<br>Add expense to the correct monthly sheet. Apply bold formatting to the data row. |
| Format Data Row | HTTP Request | Apply row formatting via batchUpdate | Build Row Format | Send Confirmation | ## 6. Add expense row<br>Add expense to the correct monthly sheet. Apply bold formatting to the data row. |
| Send Confirmation | HTTP Request | Telegram confirmation message | Format Data Row | Respond OK | ## 7. Confirm<br>Send confirmation message to user with expense details. |
| Respond OK | Respond to Webhook | End webhook (success or category-needed path) | Ask for Category / Send Confirmation | — | ## 7. Confirm<br>Send confirmation message to user with expense details. |
| Sticky Note | Sticky Note | Comment block | — | — | ## 1. Receive input<br>Receive receipt from Telegram. Check if message contains a photo or PDF. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## 2. Download file<br>Download file from Telegram servers and convert to Base64 for easybits API. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## 3. Extract data<br>Send file to easybits for extraction. Parse vendor, amount, date, and category. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## 4. Handle unknown category<br>If category is "Other", ask user to resend with a caption specifying the category. |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## 5. Prepare sheet<br>Check if monthly sheet exists. If not, create new sheet with headers and formatting. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## 6. Add expense row<br>Add expense to the correct monthly sheet. Apply bold formatting to the data row. |
| Sticky Note6 | Sticky Note | Comment block | — | — | ## 7. Confirm<br>Send confirmation message to user with expense details. |
| Sticky Note7 | Sticky Note | Comment block | — | — | ## Telegram Expense Bot<br><br>Automatically extract receipt data and organize expenses in Google Sheets.<br><br>### How it works<br>1. Send receipt photo or PDF to Telegram bot<br>2. easybits extracts vendor, amount, date, category<br>3. Data is added to the correct monthly sheet<br>4. Bot confirms the expense was logged<br><br>### How to set up<br>1. Create easybits pipeline at extractor.easybits.tech<br>2. Create Telegram bot via @BotFather<br>3. Connect Google Sheets account<br>4. Update credentials in workflow nodes<br>5. Activate and send your first receipt<br><br>### Customization<br>- Adjust reimbursement rates in "Process Extraction" node<br>- Add categories in the validCategories array<br>- Change sheet formatting in "Build Headers Request" node |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Webhook trigger**
1. Add node **Webhook** named **Telegram Webhook**  
   - Method: `POST`  
   - Path: `expense-bot`  
   - Response mode: `Using Respond to Webhook node`

2) **Validate Telegram payload contains receipt media**
2. Add **IF** node **Has Photo or PDF?**  
   - Condition (String equals):  
     - Left: `{{ $json.body.message.photo || $json.body.message.document ? 'yes' : 'no' }}`  
     - Right: `yes`
3. Connect **Telegram Webhook → Has Photo or PDF?**

3) **No-media branch**
4. Add **HTTP Request** node **No Photo Message**  
   - POST `https://api.telegram.org/bot<YOUR_TELEGRAM_BOT_TOKEN>/sendMessage`  
   - Body (JSON): include `chat_id` dynamically (recommended):  
     - `{{ $json.body.message.chat.id }}`  
   - Text: “Please send a photo of your receipt…”
5. Add **Respond to Webhook** node **Respond OK 2** with body `OK`
6. Connect **Has Photo or PDF? (false) → No Photo Message → Respond OK 2**

4) **Resolve file_id → file_path**
7. Add **HTTP Request** node **Get File Info**  
   - POST `https://api.telegram.org/bot<YOUR_TOKEN>/getFile`  
   - JSON body: `{ "file_id": "<derived>" }` using expression:  
     - `{{ $json.body.message.photo ? $json.body.message.photo.slice(-1)[0].file_id : $json.body.message.document.file_id }}`
8. Connect **Has Photo or PDF? (true) → Get File Info**

5) **Download the file**
9. Add **HTTP Request** node **Download Photo**  
   - GET `https://api.telegram.org/file/bot<YOUR_TOKEN>/{{ $json.result.file_path }}`  
   - Response: `File` (binary)
10. Connect **Get File Info → Download Photo**

6) **Convert to base64 + capture chatId**
11. Add **Code** node **Convert to Base64**  
   - Implement:
     - Detect binary key
     - Build `dataUri`
     - Extract `chatId` from Telegram Webhook node
     - Output JSON containing `dataUri`, `chatId`, `mimeType`
12. Connect **Download Photo → Convert to Base64**

7) **Send to easybits extraction**
13. Add **HTTP Request** node **Extract with easybits**  
   - POST `https://extractor.easybits.tech/api/pipelines/<YOUR_PIPELINE_ID>`  
   - Headers:
     - `Authorization: Bearer <EASYBITS_TOKEN>`
     - `Content-Type: application/json`
   - Body JSON:  
     - `{"files":["https://api.telegram.org/file/bot<YOUR_TOKEN>/{{ $('Get File Info').first().json.result.file_path }}"]}`
14. Connect **Convert to Base64 → Extract with easybits**

8) **Normalize extraction results**
15. Add **Code** node **Process Extraction**  
   - Parse extractor response into:
     - `category, vendor_name, total_amount, currency, transaction_date`
     - apply caption override from Telegram message caption
     - compute `eur_rate`, `eur_amount`
     - compute `sheet_name` as `Month_Year`
     - set `needs_category_input`
16. Connect **Extract with easybits → Process Extraction**

9) **Branch on unknown category**
17. Add **IF** node **Needs Category?**  
   - Condition: boolean true on `{{ $json.needs_category_input }}`
18. Connect **Process Extraction → Needs Category?**

10) **Ask user for category (human input path)**
19. Add **Telegram** node **Ask for Category**  
   - Credentials: create **Telegram API** credential with your bot token
   - Chat ID: `{{ $('Process Extraction').item.json.chat_id }}`
   - Text: instruct user to resend with caption `Category - description`
20. Add **Respond to Webhook** node **Respond OK** (text `OK`)
21. Connect **Needs Category? (true) → Ask for Category → Respond OK**

11) **Google Sheets: check if monthly tab exists**
22. Create **Google Sheets OAuth2** credential in n8n (Google Cloud project, OAuth consent, scopes for Sheets).
23. Add **HTTP Request** node **Check Sheet Exists**  
   - Authentication: **predefinedCredentialType** → `googleSheetsOAuth2Api`
   - GET `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>/values/'{{ sheet_name }}'!A1`
   - Options: `neverError = true`
24. Add **IF** node **Sheet Exists?**  
   - Condition: `{{ $json.values ? 'yes' : 'no' }}` equals `no`
25. Connect **Needs Category? (false) → Check Sheet Exists → Sheet Exists?**

12) **If missing: create sheet + headers**
26. Add **HTTP Request** node **Create Sheet** (Sheets batchUpdate)  
   - POST `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>:batchUpdate`
   - Body: addSheet with title `{{ sheet_name }}`
   - (Optional) set “On Error” behavior as desired (current workflow continues)
27. Add **Code** node **Build Headers Request**  
   - Read new `sheetId` from Create Sheet response
   - Build a `batchUpdate` request that:
     - writes header rows (1–3)
     - pre-formats rows 4–49
     - adds TOTAL row (row 50, formula `=SUM(G4:G49)`)
     - sets column widths
28. Add **HTTP Request** node **Setup Headers**  
   - POST `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>:batchUpdate`
   - Body: `{{ $json.requestBody }}`
29. Connect **Sheet Exists? (true) → Create Sheet → Build Headers Request → Setup Headers**

13) **If exists: retrieve sheetId**
30. Add **HTTP Request** node **Get Sheet Info**  
   - GET `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>?fields=sheets(properties(sheetId,title))`
31. Add **Code** node **Extract Sheet ID**  
   - Find `sheetId` by matching title to `{{ sheet_name }}`
32. Connect **Sheet Exists? (false) → Get Sheet Info → Extract Sheet ID**

14) **Build row payload and append**
33. Add **Code** node **Build Expense Data**  
   - Build `expenseBody.values` for append
   - Carry `sheetId` from either new-sheet path or existing-sheet path
34. Connect:
   - **Setup Headers → Build Expense Data**
   - **Extract Sheet ID → Build Expense Data**
35. Add **HTTP Request** node **Add Expense Row**  
   - POST `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>/values/{{ sheet_name }}!A4:G4:append?valueInputOption=USER_ENTERED`
   - Body: `{{ $json.expenseBody }}`
36. Connect **Build Expense Data → Add Expense Row**

15) **Format appended row**
37. Add **Code** node **Build Row Format**  
   - Parse `updatedRange` from append response to get row number
   - Build `batchUpdate` `repeatCell` request targeting that row using `sheetId`
38. Add **HTTP Request** node **Format Data Row**  
   - POST `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>:batchUpdate`
   - Body: `{{ $json.formatBody }}`
39. Connect **Add Expense Row → Build Row Format → Format Data Row**

16) **Confirm in Telegram and end webhook**
40. Add **HTTP Request** node **Send Confirmation**  
   - POST `https://api.telegram.org/bot<YOUR_TOKEN>/sendMessage`
   - chat_id: `{{ $('Process Extraction').item.json.chat_id }}`
   - text: include category/vendor/amount/date/eur_rate/eur_amount/sheet_name
41. Connect **Format Data Row → Send Confirmation → Respond OK**

17) **Set Telegram webhook**
42. In Telegram, call `setWebhook` for your bot pointing to:
- `https://<your-n8n-domain>/webhook/expense-bot` (or the production webhook URL shown by n8n)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically extract receipt data and organize expenses in Google Sheets. | Sticky Note “Telegram Expense Bot” |
| Setup steps: create easybits pipeline, create Telegram bot via @BotFather, connect Google Sheets account, update credentials, activate. | Sticky Note “Telegram Expense Bot” |
| Customization: adjust reimbursement rates in “Process Extraction”, add categories in `validCategories`, change sheet formatting in “Build Headers Request”. | Sticky Note “Telegram Expense Bot” |
| Security warning: Bot token is embedded in multiple HTTP Request URLs; prefer using Telegram node credentials or environment variables, and rotate exposed tokens. | Applies to Telegram Bot API HTTP nodes |
| Several Google Sheets URLs contain placeholders like `1EE-YOUR_AWS_SECRET_KEY_HERE` and must be corrected to a real `<SPREADSHEET_ID>`. | Affects Create Sheet / Get Sheet Info / Format Data Row |

