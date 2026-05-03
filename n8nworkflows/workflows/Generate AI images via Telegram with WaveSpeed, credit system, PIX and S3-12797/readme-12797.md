Generate AI images via Telegram with WaveSpeed, credit system, PIX and S3

https://n8nworkflows.xyz/workflows/generate-ai-images-via-telegram-with-wavespeed--credit-system--pix-and-s3-12797


# Generate AI images via Telegram with WaveSpeed, credit system, PIX and S3

## 1. Workflow Overview

**Purpose:**  
This workflow implements a Telegram bot that generates AI images (text-to-image) and edits user images (image-to-image) through **WaveSpeed** models, using a **credit-based billing system**, with **PIX payments via Mercado Pago**, and **S3-compatible storage** (AWS S3/MinIO) for uploaded images used in edits.

**Primary use cases:**
- Users chat with the bot in Telegram to:
  - Generate images from text prompts.
  - Edit an uploaded image with a caption prompt.
  - Configure generation options (aspect ratio, resolution, optional default prompt).
  - Buy credits via PIX (Brazil) and receive confirmation via webhook.

**Key logical blocks (high-level):**
1.1 **Entry + Global Configuration** (Telegram Trigger → Global env → getUser)  
1.2 **Message Classification & Status Routing** (photo/text/callback detection → status persistence → master switch)  
1.3 **Menu + Configuration Flow** (aspect ratio → resolution → optional prompt → confirmation)  
1.4 **Text-to-Image Generation Flow** (credits check → submit WaveSpeed task → poll status → deliver image / refund on failure)  
1.5 **Image Editing Flow** (upload image to S3 → build edit prompt + image URLs → credits check → submit WaveSpeed edit task → poll + deliver / refund)  
1.6 **Credits System** (deduct on submit, refund on failed tasks)  
1.7 **PIX Payments via Mercado Pago** (deposit menu → create PIX payment → send QR code → webhook validation → credit top-up)  
1.8 **Race Condition Protection for Payment Webhooks** (random wait before processing)

---

## 2. Block-by-Block Analysis

### 2.1 Entry + Global Configuration

**Overview:**  
Receives Telegram events (messages, callbacks) and loads global settings. Then attempts to fetch the user record from an n8n Data Table, continuing even if missing.

**Nodes involved:**
- **Telegram Trigger**
- **Global env**
- **getUser**
- Sticky notes: **Entry Point1**, **Global Config Info1**, **Customization Ideas1**

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — main entry point for Telegram updates.
- **Configuration (interpreted):**
  - Listens to: `message`, `callback_query`, `pre_checkout_query`
  - Download disabled (`download: false`)
- **Outputs:** to **Global env**
- **Credentials:** Telegram API credential “Imageon AI”
- **Edge cases / failures:**
  - Telegram webhook misconfiguration (URL not set / wrong token).
  - Bot not allowed in group chats (workflow later partially filters group).
  - Update payload differences between message vs callback query.

#### Node: Global env
- **Type / role:** `set` — centralized configuration object.
- **Key fields set (must be filled):**
  - `botName`, `botToken`
  - `dataTableId` (users table)
  - `paymentDataTableId` (payments table)
  - `bucketName` (S3 bucket)
  - `initialCredits` (default: `"4"`)
  - Credit costs: `generateImageCost4k` (`"0.25"`), `generateImageCost8k` (`"0.5"`)
  - Messages: `successMessage`, `taskSubmitedMessage`, `creditsNotEnough`
  - `defaultEditPrompt` (global default appended to prompts)
  - `usersPromptConfig` (boolean: true = allow prompt config step)
- **Outputs:** to **getUser**
- **Edge cases:**
  - Empty token/IDs cause downstream failures (Telegram HTTP calls, Data Table operations, S3 uploads).
- **Version notes:** `Set` node v3.4 expressions used heavily.

#### Node: getUser
- **Type / role:** `dataTable` — loads the user row using `chat_id`.
- **Configuration:**
  - Operation: **get**, Return All: **true**
  - Filter `chat_id` is derived from:
    - `callback_query.message.chat.id` OR `message.chat.id`
  - `onError: continueRegularOutput`, `alwaysOutputData: true`
- **Outputs:** to **messageType**
- **Edge cases:**
  - Missing row (first-time user) → returns empty dataset; downstream “Empty” route handles this.
  - Data type mismatch if chat_id stored numeric vs string (filters expect exact match).

---

### 2.2 Message Classification & Status Routing

**Overview:**  
Determines whether the update is a plain message, a callback, or an image upload; uploads images to S3 when needed; updates user “status” in the Data Table; routes into the correct flow based on status and message type.

**Nodes involved:**
- **messageType**
- **getFilePath**
- **downloadImage**
- **uploadS3**
- **swtichW/WOCallback**
- **switchReturn/Config**
- **upsertStatusReturn**
- **upsertStatusConfig**
- **upsertUserStatus**
- **switchMaster**
- Sticky note: **User Status Flow**, **S3 Storage**

#### Node: messageType
- **Type / role:** `switch` — classifies incoming message form (for image upload handling).
- **Outputs:**
  - “Image + Caption” → **getFilePath**
  - “Reply + Image” → **getFilePath**
  - “No Image” → **swtichW/WOCallback**
- **Logic:**
  - Checks presence of `message.photo`, and/or `message.reply_to_message.photo`.
- **Edge cases:**
  - Telegram messages can contain `document` instead of `photo` (not supported here).
  - A photo without caption still routes via photo path.

#### Node: getFilePath
- **Type / role:** `httpRequest` — calls Telegram `getFile` to retrieve file_path for the last photo size.
- **Configuration:**
  - POST `https://api.telegram.org/bot{{botToken}}/getFile`
  - JSON body includes `file_id` from either reply photo list or message photo list (last element).
- **Outputs:** to **downloadImage**
- **Failures:**
  - Invalid bot token.
  - file_id missing (expression returns undefined) → Telegram API error.

#### Node: downloadImage
- **Type / role:** `httpRequest` — downloads the image binary from Telegram file endpoint.
- **Configuration:** Response format = **file**
- **Outputs:** to **uploadS3**
- **Failures:** network timeout, Telegram file path invalid/expired.

#### Node: uploadS3
- **Type / role:** `s3` — uploads downloaded image to S3-compatible storage.
- **Configuration:**
  - Operation: upload
  - Bucket: `{{ bucketName }}`
  - Filename: `file_unique_id + '.jpg'` (from last photo)
- **Outputs:** to **swtichW/WOCallback**
- **Credentials:** S3 credential “MiniO”
- **Edge cases:**
  - The workflow assumes `.jpg` extension even if Telegram provides other formats.
  - Bucket permissions, endpoint misconfig, or missing credentials.

#### Node: swtichW/WOCallback
- **Type / role:** `switch` — decides whether to route through text commands or callback-based status updates.
- **Outputs:**
  - “W/o Callback” → **switchReturn/Config**
  - “W Callback” → **upsertUserStatus**
  - “Photo Array” → **upsertUserStatus**
- **Important condition:**
  - “W/o Callback” requires:
    - no callback_query
    - no message.photo
    - chat.type not group
- **Edge cases:**
  - Group chats largely excluded; behavior in groups may be inconsistent.

#### Node: switchReturn/Config
- **Type / role:** `switch` — handles typed commands `menu` / `config` / generic text.
- **Outputs:**
  - “Return Menu” → **upsertStatusReturn**
  - “Generate Config” → **upsertStatusConfig**
  - “Texto” → **switchMaster** (passes to status routing)
- **Edge cases:**
  - Only exact `menu` or `config` (trim + lowercase). Other languages not supported.

#### Node: upsertStatusReturn
- **Type / role:** `dataTable` — sets user status to `menu`.
- **Filter:** `chat_id = getUser.chat_id`
- **Output:** to **menuOptions**
- **Failure types:** table ID missing, no permissions.

#### Node: upsertStatusConfig
- **Type / role:** `dataTable` — sets user status to `config`.
- **Output:** to **aspectRatioConfig**

#### Node: upsertUserStatus
- **Type / role:** `dataTable` — central status updater based on callback data.
- **Behavior:**
  - If callback is an aspect ratio → status becomes `configStep2`
  - If callback is resolution:
    - if `usersPromptConfig === true` → `configStep3`
    - else → `configDone`
  - Else if callback exists → status = callback data (e.g., `generate_image`, `edit_image`, `deposit_credits`, `deposit_3`, etc.)
  - Else → keep current user status
  - Also updates:
    - `resolution` when callback is `4k` or `8k`
    - `aspect_ratio` when callback is one of allowed ratios
- **Outputs:** to **switchMaster**, **switchPromptAction**, **checkIfDepositInProgress** (fan-out)
- **Edge cases:**
  - Callback data must match exactly expected strings.
  - If user record missing, `getUser` may be empty; expressions referencing it can fail or become undefined.

#### Node: switchMaster
- **Type / role:** `switch` — routes user to the correct handler depending on stored `status`.
- **Main status routes:**
  - `menu` → **menuOptions**
  - `config` → **aspectRatioConfig**
  - `configStep2` → **resolutionConfig**
  - `configStep3` → **promptConfig**
  - `configStep3_waiting` → **upsertSavePrompt1** (expects user to send prompt text)
  - `configDone` → **newOptionAfterConfig**
  - Empty user status (first access) → **welcomeMessage**
  - `generate_image` → **messageTypeGenerateImage**
  - `edit_image` → **ifImage+Caption**
  - `deposit_credits` → **sendDepositOptions**
- **Edge cases:**
  - If status is something unexpected (typo), it may go to no output and silently stop.

---

### 2.3 First-Time User Welcome

**Overview:**  
If no user record or status exists, the bot welcomes the user, initializes their Data Table record with default settings and initial credits, then shows the menu.

**Nodes involved:**
- **welcomeMessage**
- **updateUserFirstAccess**
- **sendOptionsWelcome**

#### Node: welcomeMessage
- **Type / role:** `telegram` — sends HTML formatted intro message including initial credits.
- **Key expressions:** uses `Global env.botName` and `Global env.initialCredits`
- **Output:** to **updateUserFirstAccess**
- **Edge cases:** HTML parse errors if botName contains reserved characters (rare).

#### Node: updateUserFirstAccess
- **Type / role:** `dataTable` — upserts initial user record.
- **Sets:**
  - `status = menu`
  - `credits = initialCredits`
  - `resolution = 4k`, `aspect_ratio = 16:9`
  - counters to 0
- **Output:** to **sendOptionsWelcome**

#### Node: sendOptionsWelcome
- **Type / role:** `telegram` — shows inline keyboard menu (Generate / Edit / Deposit).
- **Output:** none (user-driven next step)

---

### 2.4 Menu + Configuration Flow (Aspect ratio → Resolution → Default prompt)

**Overview:**  
Allows users to configure aspect ratio, resolution (affecting costs), and optionally a persistent default prompt appended to every generation/edit prompt.

**Nodes involved:**
- **menuOptions**
- **aspectRatioConfig**
- **resolutionConfig**
- **promptConfig**
- **switchPromptAction**
- **upsertStatusWaitingPrompt**
- **askPromptInput1**
- **upsertSavePrompt1**
- **confirmPromptSaved1**
- **upsertClearPrompt**
- **upsertSkipPrompt**
- **newOptionAfterConfig**

#### Node: menuOptions
- **Type / role:** `telegram` — main menu message + inline keyboard.
- **Notable field:** deposit button includes `pay: true` (Telegram payment UX), but PIX here is Mercado Pago-based; Telegram payments are not actually implemented in this workflow.
- **Edge cases:** callback_data must match status strings used elsewhere.

#### Node: aspectRatioConfig
- **Type / role:** `telegram` — offers aspect ratio selection buttons.
- **Next step:** callback triggers **upsertUserStatus** setting status to `configStep2`.

#### Node: resolutionConfig
- **Type / role:** `telegram` — offers resolution selection (4k/8k) and shows costs from Global env.
- **Next step:** callback triggers **upsertUserStatus** → status becomes `configStep3` or `configDone` depending on `usersPromptConfig`.

#### Node: promptConfig
- **Type / role:** `telegram` — shows current default prompt and options to set/clear/skip.
- **Outputs:** to **switchPromptAction**
- **Formatting:** Markdown parse mode.

#### Node: switchPromptAction
- **Type / role:** `switch` — handles callback `prompt_set`, `prompt_clear`, `prompt_skip`.
- **Outputs:**
  - Set → **upsertStatusWaitingPrompt**
  - Clear → **upsertClearPrompt**
  - Skip → **upsertSkipPrompt**

#### Node: upsertStatusWaitingPrompt
- **Type / role:** `dataTable` — sets `status = configStep3_waiting`.
- **Output:** **askPromptInput1**

#### Node: askPromptInput1
- **Type / role:** `telegram` — asks user to type the default prompt (example included).
- **Edge case:** No force-reply used here; user can type anything.

#### Node: upsertSavePrompt1
- **Type / role:** `dataTable` — saves `user_default_prompt` from message text and transitions:
  - If empty text → keep `configStep3_waiting`
  - Else → `configDone`
- **Output:** **confirmPromptSaved1**
- **Edge cases:** If user sends non-text (sticker, photo), `message.text` missing → treated as empty.

#### Node: confirmPromptSaved1
- **Type / role:** `telegram` — confirms prompt saved or warns if missing.
- **Output:** **newOptionAfterConfig**

#### Node: upsertClearPrompt
- **Type / role:** `dataTable` — intended to clear stored prompt and set `status=configDone`.
- **Important issue:** It does **not** actually set `user_default_prompt` to empty/null in the shown configuration; it only includes schema for it. Result: prompt may not be cleared unless Data Table node implicitly clears missing fields (usually it doesn’t).
- **Output:** **newOptionAfterConfig**

#### Node: upsertSkipPrompt
- **Type / role:** `dataTable` — sets `status=configDone` without changing prompt.
- **Output:** **newOptionAfterConfig**

#### Node: newOptionAfterConfig
- **Type / role:** `telegram` — confirms configuration updated and shows main options.
- **Edge cases:** None significant.

---

### 2.5 Text-to-Image Generation Flow

**Overview:**  
When user status is `generate_image`, the bot validates credits, submits a WaveSpeed text-to-image task, deducts credits immediately, polls task status until completed/failed, then sends the resulting image (and refunds credits on failure).

**Nodes involved:**
- **messageTypeGenerateImage**
- **welcomeGenerateImage**
- **sendForceReply**
- **ifCreditsEnough1**
- **WaveSpeed text-to-image submit**
- **updateCreditsSendedTask**
- **taskSubmitedGenerate**
- **sendActionGenerate**
- **checkTaskGenerate**
- **switchTaskGenerate**
- **waitGenerateStatus**
- **getImage**
- **resizeImage**
- **sendGeneratedImage**
- **sendFailedGenerate**
- **updateCreditOnFail**
- **notCreditsEnoughMessage**
- Sticky note: **WaveSpeed API**, **Credit System**

#### Node: messageTypeGenerateImage
- **Type / role:** `switch` — distinguishes between:
  - Welcome via callback
  - User replying with text
  - Replying to image (to trigger edit)
  - Plain text when not in generate_image status (routes oddly; see edge cases)
- **Outputs used:**
  - “Welcome” → **welcomeGenerateImage**
  - “Reply + Text” → **ifCreditsEnough1** (submits generation)
  - “Reply + Image” → **ifCreditsEnough** (routes into edit submission path)
  - Another output “Text Prompt” also routes into **ifCreditsEnough1** (based on condition)
- **Edge cases / potential logic confusion:**
  - The “Text Prompt” condition checks `getUser.status !== 'generate_image'` while being inside generate_image flow routing; this can cause unexpected submission attempts or misrouting depending on status updates timing.

#### Node: welcomeGenerateImage
- **Type / role:** `telegram` — instructs user to write prompt; warns about queue behavior.
- **Output:** **sendForceReply**

#### Node: sendForceReply
- **Type / role:** `telegram` — forces user reply (“Now it’s your turn”).
- **Note:** It uses `chatId = {{ $json.result.chat.id }}` which assumes prior node output structure; in practice Telegram sendMessage node output often includes `result.chat.id`, but ensure this matches your n8n Telegram node version.
- **Failure risk:** if output doesn’t contain `result`, chat id expression fails.

#### Node: ifCreditsEnough1
- **Type / role:** `if` — checks credits >= cost depending on resolution.
- **Costs:** from `Global env.generateImageCost4k/8k`
- **Outputs:**
  - true → **WaveSpeed text-to-image submit**
  - false → **notCreditsEnoughMessage**

#### Node: WaveSpeed text-to-image submit
- **Type / role:** `wavespeedTaskSubmit` — submits prompt to WaveSpeed model.
- **Model:** `google/nano-banana-pro/text-to-image-ultra`
- **Parameters:**
  - `resolution = {{$json.resolution}}` and `aspect_ratio = {{$json.aspect_ratio}}` (note: relies on current item containing these; often you’ll want from `getUser`)
  - `output_format = jpeg`
  - Prompt is composed from:
    - Telegram message text
    - plus `defaultEditPrompt` (global) OR `user_default_prompt` (user) if present
- **Output:** to **updateCreditsSendedTask**
- **Failure types:**
  - WaveSpeed auth/credential errors
  - Invalid prompt (empty) if user sends blank
  - Resolution/aspect ratio missing because it references `$json.*` instead of `getUser.*` (potential bug)

#### Node: updateCreditsSendedTask
- **Type / role:** `dataTable` — deducts credits immediately after task submission.
- **Formula:** credits - (cost4k or cost8k based on user resolution)
- **Output:** to **taskSubmitedGenerate**
- **Edge cases:** if credits stored as string, Number() conversion is handled in expression; still may produce NaN if malformed.

#### Node: taskSubmitedGenerate
- **Type / role:** `telegram` — informs user task is being processed.
- **Output:** to **sendActionGenerate**

#### Node: sendActionGenerate
- **Type / role:** `telegram` — sends “upload_photo” chat action (typing indicator equivalent).
- **Output:** to **checkTaskGenerate**

#### Node: checkTaskGenerate
- **Type / role:** `wavespeedTaskStatus` — polls WaveSpeed task.
- **TaskId selection:** chooses task_id from text-to-image submit if executed; otherwise from edit submit.
- **Output:** to **switchTaskGenerate**

#### Node: switchTaskGenerate
- **Type / role:** `switch` — routes by WaveSpeed task status: completed / failed / processing / created.
- **Outputs:**
  - Completed → **getImage**
  - Failed → **sendFailedGenerate**
  - Processing/Created → **waitGenerateStatus**

#### Node: waitGenerateStatus
- **Type / role:** `wait` — pauses then loops back to polling.
- **Connection:** back to **sendActionGenerate** (which then polls again)
- **Edge cases:** without a defined wait time, defaults depend on n8n node behavior/version; ensure it’s configured to avoid rapid loops or very long waits.

#### Node: getImage
- **Type / role:** `httpRequest` — downloads the generated image from `outputs[0]`.
- **Output:** to **resizeImage**

#### Node: resizeImage
- **Type / role:** `editImage` — resizes image to 2560x1920.
- **Output:** to **sendGeneratedImage**
- **Edge cases:** if generated image isn’t a supported format, resizing fails.

#### Node: sendGeneratedImage
- **Type / role:** `telegram` sendPhoto with binary data; includes HTML caption with download link.
- **Edge cases:** Telegram caption HTML limitations; link must be valid.

#### Node: sendFailedGenerate
- **Type / role:** `telegram` — sends WaveSpeed error string.
- **Output:** to **updateCreditOnFail**

#### Node: updateCreditOnFail
- **Type / role:** `dataTable` — refunds the deducted credits on failure.
- **Refund base:** uses `updateCreditsSendedTask.item.json.credits` + cost
- **Edge case:** if `updateCreditsSendedTask` didn’t run (failure earlier), refund calculation breaks.

#### Node: notCreditsEnoughMessage
- **Type / role:** `telegram` — shows insufficient credits + current balance.
- **Text:** uses `Global env.creditsNotEnough`

---

### 2.6 Image Editing Flow (Upload → Build edit payload → Submit → Poll)

**Overview:**  
When user is in `edit_image`, the bot expects an image and caption. It uploads the image to S3 (done earlier in the shared upload block), then constructs an array of image URLs (user image + optional watermark/logo), builds prompt with defaults, checks credits, submits WaveSpeed edit task, and then uses the same polling/delivery pipeline.

**Nodes involved:**
- **ifImage+Caption**
- **welcomeImageEdit**
- **ifCreditsEnough2**
- **editFields**
- **editImageOnReply**
- **ifCreditsEnough** (used when coming from generate flow with reply+image)
- plus shared generation nodes from section 2.5 (polling, refund, send)

#### Node: ifImage+Caption
- **Type / role:** `if` — checks if current message contains `message.photo`.
- **Outputs:**
  - true → **ifCreditsEnough2** (proceed edit)
  - false → **welcomeImageEdit** (ask user to send image)
- **Edge cases:** if user sends image as document, it’s not detected.

#### Node: welcomeImageEdit
- **Type / role:** `telegram` — instructions to send photo with caption (edit prompt).
- **Output:** none

#### Node: ifCreditsEnough2
- **Type / role:** `if` — same credit check logic as generate.
- **Outputs:**
  - true → **editFields**
  - false → **notCreditsEnoughMessage**

#### Node: editFields
- **Type / role:** `set` — constructs fields required by WaveSpeed edit submission.
- **Key assignments:**
  - `photo`: builds an array of image URLs:
    - User image URL in S3:  
      `https://s3.doras.space/api/v1/buckets/{bucketName}/objects/download?...&prefix={file_unique_id}.jpg...`
    - Optional `logoOrWatermarkUrl` from Global env (note: Global env node shown does **not** define `logoOrWatermarkUrl`; this is a missing config field).
  - `message.caption`: builds prompt from:
    - user caption or text
    - + global default prompt or user default prompt
- **Output:** to **editImageOnReply**
- **Edge cases:**
  - S3 base URL is hardcoded to `https://s3.doras.space/...` rather than derived from config; if you use AWS S3/MinIO elsewhere, this URL may be wrong.
  - If the image wasn’t uploaded successfully, the referenced object won’t exist.
  - Missing `logoOrWatermarkUrl` may be undefined (handled by `if (logo)` check).

#### Node: editImageOnReply
- **Type / role:** `wavespeedTaskSubmit` — submits image-to-image edit.
- **Model:** `google/nano-banana-pro/edit-ultra`
- **Required:**
  - `images = {{$json.photo}}` (array/string; configured as “string” in schema but used as collection)
  - `prompt = {{$json.message.caption}}`
- **Optional:**
  - resolution/aspect ratio from `getUser`
- **Output:** to **updateCreditsSendedTask** (shared deduction + polling pipeline)
- **Failure types:** invalid image URLs, WaveSpeed error, auth.

#### Node: ifCreditsEnough
- **Type / role:** `if` — used when the user triggers edit while inside generation flow (replying with image).
- **True output:** **editFields**, false: **notCreditsEnoughMessage**

---

### 2.7 PIX Deposit Flow (User-facing)

**Overview:**  
Lets users choose a deposit amount (R$3/6/10). Prevents starting a new deposit if one is already in progress. Creates a Mercado Pago PIX payment, stores a pending payment record, and sends the QR code to the user in Telegram.

**Nodes involved:**
- **sendDepositOptions**
- **checkIfDepositInProgress**
- **qrCodePaymentInProgress**
- **switchDeposit**
- **setDepositAmount3 / 6 / 10**
- **depositCredentials**
- **insertPaymentRow**
- **apiMercadoPago**
- **getPix**
- **pix_baseToFile**
- **sendPixQRCode**
- **sendPixText**
- Sticky note: **Payment Integration**

#### Node: sendDepositOptions
- **Type / role:** `telegram` — shows deposit amount buttons.
- **Next:** callback updates status via **upsertUserStatus**, then **checkIfDepositInProgress** evaluates.

#### Node: checkIfDepositInProgress
- **Type / role:** `if` — checks if user status is already one of `deposit_3`, `deposit_6`, `deposit_10`.
- **Outputs:**
  - true → **qrCodePaymentInProgress**
  - false → **switchDeposit**
- **Edge case:** This logic considers being in any deposit_* status as “in progress”, but doesn’t verify whether a pending payment exists in the payments table.

#### Node: qrCodePaymentInProgress
- **Type / role:** `telegram` — warns user they already have a QR code awaiting payment.

#### Node: switchDeposit
- **Type / role:** `switch` — routes based on `$json.status` being `deposit_3`, `deposit_6`, or `deposit_10`.
- **Outputs:** to corresponding setDepositAmount node.
- **Edge case:** No explicit renameOutput keys shown for first rule; still works but less readable.

#### Nodes: setDepositAmount3 / setDepositAmount6 / setDepositAmount10
- **Type / role:** `set` — sets `preco` to `"3"`, `"6"`, or `"10"`.
- **Outputs:** to **depositCredentials**

#### Node: depositCredentials
- **Type / role:** `set` — prepares Mercado Pago request fields.
- **Critical fields to fill:**
  - `public_Key` (unused in request)
  - `access_token` (used in Authorization)
  - `notification_url` (must be your public webhook URL pointing to **paymentStatus**)
  - `cpf`, `email_cliente` (currently hardcoded placeholders)
- **Generates:** `external_reference = 'CLIENTE' + chat_id + '-' + random4digits`
- **Output:** to **insertPaymentRow**
- **Edge cases / compliance:**
  - CPF/email should be real user payer data; storing/handling PII requires care.
  - Notification URL must be reachable by Mercado Pago.

#### Node: insertPaymentRow
- **Type / role:** `dataTable` — inserts a pending payment row.
- **Table:** `paymentDataTableId` from Global env
- **Fields:** amount, status=pending, chat_id, reference (external_reference)
- **Output:** to **apiMercadoPago**

#### Node: apiMercadoPago
- **Type / role:** `httpRequest` — creates Mercado Pago PIX payment.
- **Headers:**
  - `Authorization: Bearer {access_token}`
  - `X-Idempotency-Key: {external_reference}`
- **Body:** description, external_reference, notification_url, payer(email+CPF), payment_method_id=pix, transaction_amount
- **Output:** **getPix**
- **Failure types:**
  - Invalid access token
  - payer identification invalid
  - Mercado Pago API downtime

#### Node: getPix
- **Type / role:** `set` — extracts:
  - `pix_url` (copy/paste code)
  - `pix_base64` (QR image)
  - `id_transacao`
- **Output:** to **pix_baseToFile**

#### Node: pix_baseToFile
- **Type / role:** `convertToFile` — converts base64 string to binary data for Telegram photo sending.
- **Output:** to **sendPixQRCode**

#### Node: sendPixQRCode
- **Type / role:** `telegram` sendPhoto — sends the QR image with caption containing PIX copy/paste code.
- **Output:** to **sendPixText**

#### Node: sendPixText
- **Type / role:** `telegram` — sends follow-up instructions.

---

### 2.8 Payment Webhook Processing + Credit Top-up

**Overview:**  
Receives Mercado Pago webhook calls, applies a random delay to reduce concurrency issues, fetches payment status from Mercado Pago, validates payment reference record, updates payment row, updates user credits, and notifies the user.

**Nodes involved:**
- **paymentStatus**
- **raceConditionDelay**
- **getPaymentStatus**
- **fetchPaymentRecord**
- **switchForPaymentStatus**
- **checkIfPaymentPending**
- **upsertPaymentStatus**
- **getUserAfterPayment**
- **updateUserCreditsAfterPayment**
- **paymentConfirmedText**
- **sendOptionsAfterPayment**
- **getUserAfterDecline**
- **updateUserStatusAfterDecline**
- **paymentDeclinedText**
- Sticky notes: **Race Condition Info**, **Pending Payment Validation**

#### Node: paymentStatus
- **Type / role:** `webhook` — receives Mercado Pago notifications.
- **Path:** `your-webhook-path` (must match notification_url)
- **Output:** to **raceConditionDelay**
- **Edge cases:** Mercado Pago may send different fields (`id`, `data.id`). Workflow attempts both later.

#### Node: raceConditionDelay
- **Type / role:** `wait` — random delay 1–4 seconds.
- **Purpose:** reduce simultaneous processing collisions.
- **Output:** to **getPaymentStatus**

#### Node: getPaymentStatus
- **Type / role:** `httpRequest` — fetch Mercado Pago payment details by id.
- **URL:** `https://api.mercadopago.com/v1/payments/{id}`
  - id from `paymentStatus.query.id` OR `paymentStatus.query['data.id']`
- **Header Authorization:** currently set to `"="` (blank). This must be set to `Bearer <access_token>` or use credentials.
- **Output:** to **fetchPaymentRecord**
- **Critical issue:** As configured, this request will fail unless Authorization is filled.

#### Node: fetchPaymentRecord
- **Type / role:** `dataTable` — retrieves stored payment row by `reference = external_reference`.
- **Table ID:** hardcoded `SsrqjZWPYOVsFA2Q` (not taken from Global env), which may be wrong in other environments.
- **Output:** to **switchForPaymentStatus**
- **Edge cases:** reference not found → downstream should stop, but workflow doesn’t have an explicit “not found” guard besides checking pending status.

#### Node: switchForPaymentStatus
- **Type / role:** `switch` — routes by Mercado Pago payment status:
  - approved → **checkIfPaymentPending**
  - pending → (no connected action)
  - rejected → **getUserAfterDecline**
- **Edge cases:** other statuses (cancelled, refunded, etc.) not handled.

#### Node: checkIfPaymentPending
- **Type / role:** `if` — checks if the stored payment record is still `pending` to prevent duplicates.
- **True:** **upsertPaymentStatus**
- **False:** stops (no output connected)

#### Node: upsertPaymentStatus
- **Type / role:** `dataTable` — sets payment row `status=approved`.
- **Table:** hardcoded `SsrqjZWPYOVsFA2Q`
- **Output:** **getUserAfterPayment**

#### Node: getUserAfterPayment
- **Type / role:** `dataTable` — loads user (but tableId is also hardcoded to `SsrqjZWPYOVsFA2Q`, which looks wrong; it should be users table ID).
- **This is likely a configuration bug:** it uses same ID as payment table.
- **Output:** **updateUserCreditsAfterPayment**

#### Node: updateUserCreditsAfterPayment
- **Type / role:** `dataTable` — sets user `status=menu` and increases credits by `total_paid_amount`.
- **Again tableId is hardcoded** and likely incorrect.
- **Output:** **paymentConfirmedText**
- **Edge cases:** currency mismatch — credits are treated like currency (R$) in messages; generation costs are fractional credits. Ensure consistent unit.

#### Node: paymentConfirmedText
- **Type / role:** `telegram` — confirms payment and shows current balance formatted as BRL.
- **Output:** **sendOptionsAfterPayment**

#### Node: sendOptionsAfterPayment
- **Type / role:** `telegram` — shows options after payment.
- **Bug:** callback data includes `deposti_credits` (typo) instead of `deposit_credits`, so deposit button may not work.
- **Credentials:** Telegram credential “Telegram Loja” (different bot/credential from main “Imageon AI”)
- **Risk:** sending messages from a different bot token to the same chat may fail.

#### Decline path nodes
- **getUserAfterDecline** → **updateUserStatusAfterDecline** → **paymentDeclinedText** → **sendOptionsAfterPayment**
- **Important issue:** `updateUserStatusAfterDecline` and `paymentDeclinedText` reference `getFieldsChatId`, which does not exist in enabled nodes (likely leftover). This will break decline handling unless corrected.

---

### 2.9 Disabled Demo Flows

**Overview:**  
There are disabled flows intended to send demo media on first access and demo edit examples.

**Nodes involved (all disabled):**
- **If demo sended1**
- **Welcome**
- **Send welcome media**
- **Update welcome_demo_sended**
- **Fields Chat**
- **If use_edit_demo1**
- **Welcome image edit message**
- **Send a media group demo1**
- **Update use_edit_demo**
- **Edit image question**
- Sticky notes: **Demo Welcome Flow1**, **Demo Edit Image Flow1**

**Notes / risks:**  
Disabled nodes are still part of the workflow JSON and should be documented for completeness. If re-enabled, verify their credentials (“Telegram Loja”) and data table IDs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | stickyNote | Explains workflow purpose and setup | — | — | (content includes setup steps) |
| Telegram Trigger | telegramTrigger | Entry point for Telegram updates | — | Global env | 🎯 **Entry Point**  Telegram Trigger receives all messages and callbacks. Routes to appropriate handlers. |
| Global env | set | Central config variables | Telegram Trigger | getUser | 🌍 **Global Environment**  Main bot configuration: tokens, database IDs, credit costs, and default messages. Edit carefully. |
| getUser | dataTable | Fetch user by chat_id | Global env | messageType | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| messageType | switch | Detect message image/reply mode | getUser | getFilePath / swtichW/WOCallback | 📦 **S3 Storage**  Stores uploaded images for AI editing. Compatible with AWS S3 or MinIO. |
| getFilePath | httpRequest | Telegram getFile call | messageType | downloadImage | 📦 **S3 Storage**  Stores uploaded images for AI editing. Compatible with AWS S3 or MinIO. |
| downloadImage | httpRequest | Download Telegram file binary | getFilePath | uploadS3 | 📦 **S3 Storage**  Stores uploaded images for AI editing. Compatible with AWS S3 or MinIO. |
| uploadS3 | s3 | Upload image to S3 bucket | downloadImage | swtichW/WOCallback | 📦 **S3 Storage**  Stores uploaded images for AI editing. Compatible with AWS S3 or MinIO. |
| swtichW/WOCallback | switch | Route callback vs normal message | messageType / uploadS3 | switchReturn/Config / upsertUserStatus | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| switchReturn/Config | switch | Handle typed “menu”/“config” | swtichW/WOCallback | upsertStatusReturn / upsertStatusConfig / switchMaster | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| upsertStatusReturn | dataTable | Set status=menu | switchReturn/Config | menuOptions |  |
| upsertStatusConfig | dataTable | Set status=config | switchReturn/Config | aspectRatioConfig |  |
| upsertUserStatus | dataTable | Update status based on callback | swtichW/WOCallback | switchMaster / switchPromptAction / checkIfDepositInProgress | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| switchMaster | switch | Route by user status | upsertUserStatus / switchReturn/Config | menuOptions / aspectRatioConfig / resolutionConfig / promptConfig / upsertSavePrompt1 / newOptionAfterConfig / welcomeMessage / messageTypeGenerateImage / ifImage+Caption / sendDepositOptions | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| welcomeMessage | telegram | First-time welcome text | switchMaster | updateUserFirstAccess |  |
| updateUserFirstAccess | dataTable | Create/init user record | welcomeMessage | sendOptionsWelcome |  |
| sendOptionsWelcome | telegram | Menu after welcome | updateUserFirstAccess | — |  |
| menuOptions | telegram | Main menu | switchMaster / upsertStatusReturn | — |  |
| aspectRatioConfig | telegram | Config step 1: aspect ratio | switchMaster / upsertStatusConfig | — |  |
| resolutionConfig | telegram | Config step 2: resolution | switchMaster | — |  |
| promptConfig | telegram | Config step 3: prompt options | switchMaster | switchPromptAction |  |
| switchPromptAction | switch | Handle prompt_set/clear/skip | upsertUserStatus / promptConfig | upsertStatusWaitingPrompt / upsertClearPrompt / upsertSkipPrompt |  |
| upsertStatusWaitingPrompt | dataTable | Set status=configStep3_waiting | switchPromptAction | askPromptInput1 |  |
| askPromptInput1 | telegram | Ask user to type default prompt | upsertStatusWaitingPrompt | — |  |
| upsertSavePrompt1 | dataTable | Save user_default_prompt from text | switchMaster | confirmPromptSaved1 |  |
| confirmPromptSaved1 | telegram | Confirm prompt saved | upsertSavePrompt1 | newOptionAfterConfig |  |
| upsertClearPrompt | dataTable | Mark config done (intended prompt clear) | switchPromptAction | newOptionAfterConfig |  |
| upsertSkipPrompt | dataTable | Skip prompt, config done | switchPromptAction | newOptionAfterConfig |  |
| newOptionAfterConfig | telegram | Config updated + menu options | switchMaster / confirmPromptSaved1 / upsertClearPrompt / upsertSkipPrompt | — |  |
| messageTypeGenerateImage | switch | Handle generation sub-states | switchMaster | ifCreditsEnough1 / welcomeGenerateImage / ifCreditsEnough | 🎨 **WaveSpeed API**  AI image generation and editing. Models: nano-banana-pro for text-to-image and edit-ultra. |
| welcomeGenerateImage | telegram | Explain prompt entry for generation | messageTypeGenerateImage | sendForceReply |  |
| sendForceReply | telegram | Force reply prompt input | welcomeGenerateImage | — |  |
| ifCreditsEnough1 | if | Credits check for generation | messageTypeGenerateImage | WaveSpeed text-to-image submit / notCreditsEnoughMessage | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| WaveSpeed text-to-image submit | wavespeedTaskSubmit | Submit text-to-image task | ifCreditsEnough1 | updateCreditsSendedTask | 🎨 **WaveSpeed API**  AI image generation and editing. Models: nano-banana-pro for text-to-image and edit-ultra. |
| updateCreditsSendedTask | dataTable | Deduct credits on submit | WaveSpeed text-to-image submit / editImageOnReply | taskSubmitedGenerate | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| taskSubmitedGenerate | telegram | Notify task queued | updateCreditsSendedTask | sendActionGenerate |  |
| sendActionGenerate | telegram | Send chat action “upload_photo” | taskSubmitedGenerate / waitGenerateStatus | checkTaskGenerate |  |
| checkTaskGenerate | wavespeedTaskStatus | Poll WaveSpeed task | sendActionGenerate | switchTaskGenerate | 🎨 **WaveSpeed API**  AI image generation and editing. Models: nano-banana-pro for text-to-image and edit-ultra. |
| switchTaskGenerate | switch | Route by task status | checkTaskGenerate | getImage / sendFailedGenerate / waitGenerateStatus |  |
| waitGenerateStatus | wait | Delay before re-polling | switchTaskGenerate | sendActionGenerate |  |
| getImage | httpRequest | Download generated image | switchTaskGenerate | resizeImage |  |
| resizeImage | editImage | Resize output for Telegram | getImage | sendGeneratedImage |  |
| sendGeneratedImage | telegram | Send final image + link | resizeImage | — |  |
| sendFailedGenerate | telegram | Send failure reason | switchTaskGenerate | updateCreditOnFail |  |
| updateCreditOnFail | dataTable | Refund credits on failure | sendFailedGenerate | — | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| notCreditsEnoughMessage | telegram | Inform user insufficient credits | ifCreditsEnough1 / ifCreditsEnough2 / ifCreditsEnough | — | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| ifImage+Caption | if | Edit mode: check for photo | switchMaster | ifCreditsEnough2 / welcomeImageEdit |  |
| welcomeImageEdit | telegram | Ask user to send photo + caption | ifImage+Caption | — |  |
| ifCreditsEnough2 | if | Credits check for edit | ifImage+Caption | editFields / notCreditsEnoughMessage | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| ifCreditsEnough | if | Credits check for edit (from generate flow) | messageTypeGenerateImage | editFields / notCreditsEnoughMessage | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| editFields | set | Build edit images array + prompt | ifCreditsEnough2 / ifCreditsEnough | editImageOnReply |  |
| editImageOnReply | wavespeedTaskSubmit | Submit image-to-image edit task | editFields | updateCreditsSendedTask | 🎨 **WaveSpeed API**  AI image generation and editing. Models: nano-banana-pro for text-to-image and edit-ultra. |
| sendDepositOptions | telegram | Deposit menu amounts | switchMaster | — | 💰 **PIX Payment**  Brazilian instant payment via Mercado Pago API. Handles QR code generation and webhook confirmations. |
| checkIfDepositInProgress | if | Prevent concurrent deposits | upsertUserStatus | qrCodePaymentInProgress / switchDeposit | 💰 **PIX Payment**  Brazilian instant payment via Mercado Pago API. Handles QR code generation and webhook confirmations. |
| qrCodePaymentInProgress | telegram | Warn deposit already in progress | checkIfDepositInProgress | — | 💰 **PIX Payment**  Brazilian instant payment via Mercado Pago API. Handles QR code generation and webhook confirmations. |
| switchDeposit | switch | Route chosen deposit amount | checkIfDepositInProgress | setDepositAmount3/6/10 | 💰 **PIX Payment**  Brazilian instant payment via Mercado Pago API. Handles QR code generation and webhook confirmations. |
| setDepositAmount3 | set | Set preco=3 | switchDeposit | depositCredentials |  |
| setDepositAmount6 | set | Set preco=6 | switchDeposit | depositCredentials |  |
| setDepositAmount10 | set | Set preco=10 | switchDeposit | depositCredentials |  |
| depositCredentials | set | Prepare Mercado Pago fields | setDepositAmount3/6/10 | insertPaymentRow |  |
| insertPaymentRow | dataTable | Store pending payment | depositCredentials | apiMercadoPago |  |
| apiMercadoPago | httpRequest | Create PIX payment | insertPaymentRow | getPix |  |
| getPix | set | Extract QR + code | apiMercadoPago | pix_baseToFile |  |
| pix_baseToFile | convertToFile | Convert base64 to binary | getPix | sendPixQRCode |  |
| sendPixQRCode | telegram | Send QR code image | pix_baseToFile | sendPixText |  |
| sendPixText | telegram | Send instructions | sendPixQRCode | — |  |
| paymentStatus | webhook | Mercado Pago webhook receiver | — | raceConditionDelay | ✅ **Payment Validation**  Checks if payment reference exists before processing webhook to avoid duplicates. |
| raceConditionDelay | wait | Random delay 1–4s | paymentStatus | getPaymentStatus | 🕒 **Race Condition Protection**  Random delay (1-4s) prevents conflicts from simultaneous webhook executions. |
| getPaymentStatus | httpRequest | Fetch payment details from MP | raceConditionDelay | fetchPaymentRecord | ✅ **Payment Validation**  Checks if payment reference exists before processing webhook to avoid duplicates. |
| fetchPaymentRecord | dataTable | Find payment record by reference | getPaymentStatus | switchForPaymentStatus | ✅ **Payment Validation**  Checks if payment reference exists before processing webhook to avoid duplicates. |
| switchForPaymentStatus | switch | Route approved/pending/rejected | fetchPaymentRecord | checkIfPaymentPending / getUserAfterDecline |  |
| checkIfPaymentPending | if | Ensure not already processed | switchForPaymentStatus | upsertPaymentStatus |  |
| upsertPaymentStatus | dataTable | Mark payment approved | checkIfPaymentPending | getUserAfterPayment |  |
| getUserAfterPayment | dataTable | Load user for credit update | upsertPaymentStatus | updateUserCreditsAfterPayment |  |
| updateUserCreditsAfterPayment | dataTable | Add credits and set menu | getUserAfterPayment | paymentConfirmedText |  |
| paymentConfirmedText | telegram | Notify user payment confirmed | updateUserCreditsAfterPayment | sendOptionsAfterPayment |  |
| sendOptionsAfterPayment | telegram | Menu after payment | paymentConfirmedText / paymentDeclinedText | — |  |
| getUserAfterDecline | dataTable | Load user after reject | switchForPaymentStatus | updateUserStatusAfterDecline |  |
| updateUserStatusAfterDecline | dataTable | Reset status menu after reject | getUserAfterDecline | paymentDeclinedText |  |
| paymentDeclinedText | telegram | Notify declined payment | updateUserStatusAfterDecline | sendOptionsAfterPayment |  |
| Credit System | stickyNote | Commentary | — | — | 💳 **Credit System**  Users get initial credits, deducted on generation. Failed tasks refund credits automatically. |
| Payment Integration | stickyNote | Commentary | — | — | 💰 **PIX Payment**  Brazilian instant payment via Mercado Pago API. Handles QR code generation and webhook confirmations. |
| S3 Storage | stickyNote | Commentary | — | — | 📦 **S3 Storage**  Stores uploaded images for AI editing. Compatible with AWS S3 or MinIO. |
| WaveSpeed API | stickyNote | Commentary | — | — | 🎨 **WaveSpeed API**  AI image generation and editing. Models: nano-banana-pro for text-to-image and edit-ultra. |
| User Status Flow | stickyNote | Commentary | — | — | 🔄 **User Status**  Tracks user state: menu, config steps, generate_image, edit_image, deposit_credits. |
| Race Condition Info | stickyNote | Commentary | — | — | 🕒 **Race Condition Protection**  Random delay (1-4s) prevents conflicts from simultaneous webhook executions. |
| Pending Payment Validation | stickyNote | Commentary | — | — | ✅ **Payment Validation**  Checks if payment reference exists before processing webhook to avoid duplicates. |
| Demo Welcome Flow1 | stickyNote | Disabled commentary | — | — | Welcome flow with demo media for first-time users. Currently disabled. |
| Demo Edit Image Flow1 | stickyNote | Disabled commentary | — | — | Demo flow showing image editing capabilities. Currently disabled. |
| (All disabled demo nodes) | (various) | Disabled demo features | — | — | (see the two demo sticky notes) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create required Data Tables**
   1. Create a **Users** Data Table with columns (minimum):
      - `chat_id` (string)
      - `status` (string)
      - `credits` (string/number; choose one and keep consistent)
      - `resolution` (string: `4k|8k`)
      - `aspect_ratio` (string)
      - `user_default_prompt` (string, optional)
      - `number_images` (number, optional)
      - `number_videos` (number, optional)
   2. Create a **Payments** Data Table with columns:
      - `reference` (string)
      - `chat_id` (string)
      - `amount` (number)
      - `status` (string: pending/approved/…)
   3. Copy the table IDs to use in the workflow.

2) **Create Telegram bot + credentials**
   1. Create bot with **@BotFather**, get token.
   2. In n8n, create **Telegram API credential** using the token.
   3. Prepare webhook base URL for n8n (publicly accessible).

3) **Create S3/MinIO credentials**
   1. Add S3 credentials (access key/secret, endpoint for MinIO if applicable).
   2. Create bucket and note bucket name.

4) **Create Mercado Pago credentials**
   1. Generate Mercado Pago **access token**.
   2. Decide the webhook URL for payment notifications (n8n Webhook node public URL).

5) **Build workflow nodes (core path)**
   1. Add **Telegram Trigger**
      - Updates: message, callback_query, pre_checkout_query
   2. Add **Set** node named **Global env**
      - Add fields: botName, botToken, dataTableId, paymentDataTableId, bucketName, initialCredits, generateImageCost4k, generateImageCost8k, messages, defaultEditPrompt, usersPromptConfig.
      - Connect: Telegram Trigger → Global env
   3. Add **Data Table** node **getUser**
      - Operation: get, returnAll true
      - Filter chat_id from callback_query.message.chat.id OR message.chat.id
      - On error: continue regular output; always output data
      - Connect: Global env → getUser
   4. Add **Switch** node **messageType**
      - Routes for Image+Caption, Reply+Image, No Image (as in JSON)
   5. Add Telegram file download chain:
      - **HTTP Request** getFilePath (Telegram getFile)
      - **HTTP Request** downloadImage (Telegram file)
      - **S3** uploadS3 (upload binary to bucket, filename = file_unique_id.jpg)
      - Connect: messageType → getFilePath → downloadImage → uploadS3
   6. Add **Switch** swtichW/WOCallback
      - Outputs: W/o Callback, W Callback, Photo Array
      - Connect: uploadS3 → swtichW/WOCallback AND messageType “No Image” → swtichW/WOCallback
   7. Add **Switch** switchReturn/Config
      - Detect message.text == menu / config / any text
      - Connect: swtichW/WOCallback (W/o Callback) → switchReturn/Config
   8. Add **Data Table** upserts:
      - upsertStatusReturn (status=menu)
      - upsertStatusConfig (status=config)
      - Connect switchReturn/Config accordingly
   9. Add **Data Table** upsertUserStatus
      - Set status based on callback_query.data rules (aspect ratio, resolution, other callbacks)
      - Update resolution/aspect_ratio when relevant
      - Connect: swtichW/WOCallback (W Callback + Photo Array) → upsertUserStatus
   10. Add **Switch** switchMaster (status router)
       - Outputs: menu/config/configStep2/configStep3/configStep3_waiting/configDone/Empty/generate_image/edit_image/deposit_credits
       - Connect: switchReturn/Config “Texto” → switchMaster
       - Connect: upsertUserStatus → switchMaster

6) **Add welcome/init flow**
   1. Telegram **welcomeMessage** (HTML)
   2. DataTable **updateUserFirstAccess** (upsert defaults + initial credits)
   3. Telegram **sendOptionsWelcome**
   4. Connect: switchMaster “Empty” → welcomeMessage → updateUserFirstAccess → sendOptionsWelcome

7) **Add menu + config flow**
   1. Telegram **menuOptions** (inline keyboard)
   2. Telegram **aspectRatioConfig**
   3. Telegram **resolutionConfig**
   4. Telegram **promptConfig** + **switchPromptAction**
   5. DataTable updates:
      - upsertStatusWaitingPrompt → Telegram askPromptInput1
      - upsertSavePrompt1 → Telegram confirmPromptSaved1 → Telegram newOptionAfterConfig
      - upsertSkipPrompt / upsertClearPrompt → newOptionAfterConfig
   6. Connect outputs from switchMaster accordingly.

8) **Add generation flow**
   1. Switch **messageTypeGenerateImage**
   2. Telegram **welcomeGenerateImage** → **sendForceReply**
   3. IF **ifCreditsEnough1**
   4. WaveSpeed **WaveSpeed text-to-image submit** (set WaveSpeed credential)
   5. DataTable **updateCreditsSendedTask**
   6. Telegram **taskSubmitedGenerate** → **sendActionGenerate**
   7. WaveSpeed **checkTaskGenerate** → **switchTaskGenerate**
   8. Wait **waitGenerateStatus** loop back to sendActionGenerate
   9. On completed: HTTP **getImage** → **resizeImage** → Telegram **sendGeneratedImage**
   10. On failed: Telegram **sendFailedGenerate** → DataTable **updateCreditOnFail**

9) **Add edit flow**
   1. IF **ifImage+Caption** (checks message.photo)
   2. Telegram **welcomeImageEdit** (else branch)
   3. IF **ifCreditsEnough2**
   4. Set **editFields** to build:
      - S3 download URL for the uploaded file_unique_id.jpg
      - prompt composition logic
   5. WaveSpeed **editImageOnReply**
   6. Reuse the same deduction + polling pipeline (updateCreditsSendedTask etc.)

10) **Add deposit (PIX) flow**
   1. Telegram **sendDepositOptions**
   2. IF **checkIfDepositInProgress** → Telegram **qrCodePaymentInProgress**
   3. Switch **switchDeposit** → Set **setDepositAmount3/6/10**
   4. Set **depositCredentials**:
      - Fill access_token, notification_url, payer CPF/email (ideally user-provided)
   5. DataTable **insertPaymentRow** (paymentDataTableId)
   6. HTTP **apiMercadoPago** (Authorization Bearer token + idempotency key)
   7. Set **getPix** → ConvertToFile **pix_baseToFile**
   8. Telegram **sendPixQRCode** → Telegram **sendPixText**

11) **Add payment webhook processing**
   1. Webhook **paymentStatus** (public endpoint)
   2. Wait **raceConditionDelay** (1–4 seconds)
   3. HTTP **getPaymentStatus** (must include Authorization Bearer MP token)
   4. DataTable **fetchPaymentRecord** by external_reference
   5. Switch **switchForPaymentStatus**
   6. IF **checkIfPaymentPending**
   7. DataTable **upsertPaymentStatus** to approved
   8. DataTable **getUserAfterPayment** using the **Users table ID**
   9. DataTable **updateUserCreditsAfterPayment** (credits += total_paid_amount; status=menu)
   10. Telegram **paymentConfirmedText** → Telegram **sendOptionsAfterPayment**

12) **Fix known configuration issues while rebuilding**
   - Ensure `getPaymentStatus` Authorization header is set.
   - Ensure payment table ID and user table ID are not mixed/hardcoded.
   - Fix typo `deposti_credits` → `deposit_credits`.
   - Fix decline path references to `getFieldsChatId` (either create that node or use `$json.chat_id`).
   - In WaveSpeed text-to-image submit, consider using `getUser.resolution` and `getUser.aspect_ratio` reliably.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This Telegram bot enables AI-powered image generation and editing with a built-in credit system… handles concurrent requests safely with race condition protection.” | From **Main Overview** sticky note |
| Setup steps include: create Telegram bot, WaveSpeed credentials, S3 storage, n8n Data Table with columns, configure Global env, set up credentials, update webhook URLs, activate workflow | From **Main Overview** sticky note |
| Credit costs differ by resolution (4K/8K) and credits are deducted on submission then refunded on failed tasks | Credit System behavior implemented across IF + Data Table nodes |
| PIX payments are Mercado Pago-based, not Telegram Payments (despite “pay: true” in one menu button) | Menu configuration vs actual Mercado Pago integration |
| S3 download URL in editFields is hardcoded to `https://s3.doras.space/...` and should be parameterized for portability | Edit flow implementation detail |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.