Track meal nutrition from meal photos with LINE, Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/track-meal-nutrition-from-meal-photos-with-line--google-gemini-and-google-sheets-12380


# Track meal nutrition from meal photos with LINE, Google Gemini and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI Meal Nutrition Tracker with LINE and Google Sheets  
**Purpose:** Receive meal photos sent via LINE, download the image, analyze it with Google Gemini (vision + text), extract structured nutrition metrics, store the image in Google Drive, log the meal in Google Sheets, and push a formatted analysis message back to the user on LINE. If the image is not food-related, send a fallback message.

### 1.1 Input Reception & Configuration
- Entry point is a **LINE webhook**.
- A **Config** Set node stores tokens and IDs used throughout the flow.

### 1.2 Validation & Image Retrieval
- Checks whether the incoming LINE message is an **image**.
- Downloads the binary image content from LINE’s data API.

### 1.3 AI Analysis (Gemini Vision)
- Sends the image to **Google Gemini** with a strict, line-based key/value output format.
- Rejects results that contain the sentinel phrase **“Not a food image”**.

### 1.4 Parsing, Storage, and Logging
- Parses Gemini text into structured fields (calories/macros/score/advice).
- Merges parsed JSON with the original binary image.
- Uploads the image to Google Drive, then appends a row to Google Sheets.

### 1.5 User Response
- Pushes a detailed formatted nutrition summary back to the LINE user.
- Alternative branch: notify user if the image is not food.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Configuration
**Overview:** Receives LINE events and initializes configuration variables (tokens, IDs, calorie goal) used in downstream nodes.

**Nodes involved:**
- LINE Webhook
- Config

#### Node: LINE Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow trigger receiving HTTP POST from LINE.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `/MealTracker`
  - **WebhookId (internal):** `meal-tracker-webhook`
- **Inputs/Outputs:**
  - **Output →** Config
- **Expressions/variables used downstream:**
  - `$('LINE Webhook').item.json.body.events[0]...` is referenced in many nodes for message id and userId.
- **Failure/edge cases:**
  - LINE sends different event types (follow/unfollow/postback). This workflow assumes `body.events[0].message` exists.
  - Signature verification is not implemented (LINE usually uses `X-Line-Signature` header); without verification, endpoint can be abused.
  - If multiple events are batched, only `[0]` is used.
- **Version notes:** Node typeVersion `2.1` (n8n webhook options differ slightly across versions).

#### Node: Config
- **Type / role:** `Set` (n8n-nodes-base.set) — centralizes “constants” in one place.
- **Key configuration choices:**
  - Sets fields:
    - `LINE_CHANNEL_ACCESS_TOKEN` (string placeholder)
    - `GOOGLE_SHEETS_ID` (string placeholder)
    - `GOOGLE_DRIVE_FOLDER_ID` (string placeholder)
    - `DAILY_CALORIE_GOAL` (number, default 2000)
  - Uses “assignments” mode (define fields explicitly).
- **Inputs/Outputs:**
  - **Input ←** LINE Webhook
  - **Output →** Check If Image
- **Failure/edge cases:**
  - Leaving placeholders will cause auth failures in LINE/Google calls.
  - If you change field names here, all downstream expressions must be updated.
- **Version notes:** typeVersion `3.4`.

---

### Block 2 — Validation & Image Retrieval
**Overview:** Ensures the inbound message is an image and downloads the image bytes from LINE so it can be analyzed and stored.

**Nodes involved:**
- Check If Image
- Download Image from LINE

#### Node: Check If Image
- **Type / role:** `Switch` (n8n-nodes-base.switch) — routes execution only if message type equals `image`.
- **Key configuration:**
  - Rule checks:
    - Left: `={{ $('LINE Webhook').item.json.body.events[0].message.type }}`
    - Equals: `"image"`
  - Only one rule is defined; the “false” path is not connected.
- **Inputs/Outputs:**
  - **Input ←** Config
  - **Output (match) →** Download Image from LINE
- **Failure/edge cases:**
  - If message is text/sticker/video, workflow stops silently (no user feedback branch is configured).
  - If `events[0].message` is missing (non-message events), expression may evaluate to `undefined` and not match.
- **Version notes:** typeVersion `3.3`.

#### Node: Download Image from LINE
- **Type / role:** `HTTP Request` (n8n-nodes-base.httpRequest) — downloads binary image content from LINE Messaging API.
- **Key configuration:**
  - **URL:**  
    `https://api-data.line.me/v2/bot/message/{{ message.id }}/content`  
    using expression:  
    `=https://api-data.line.me/v2/bot/message/{{ $('LINE Webhook').item.json.body.events[0].message.id }}/content`
  - **Headers:**
    - `Authorization: Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
  - **Response:** `responseFormat: file` (binary)
- **Inputs/Outputs:**
  - **Input ←** Check If Image
  - **Outputs →**
    - Merge Data (input 0)
    - Analyze Meal with AI (main input)
- **Failure/edge cases:**
  - 401/403 if token invalid or missing required permissions.
  - 404 if message content expired (LINE content is time-limited).
  - If LINE returns non-image content or unexpected content-type, downstream AI node may fail.
- **Version notes:** typeVersion `4.3`.

---

### Block 3 — AI Analysis (Gemini Vision) & Result Gate
**Overview:** Sends the image to Gemini with a strict output schema and decides whether to proceed (food) or notify user (not food).

**Nodes involved:**
- Google Gemini Chat Model
- Analyze Meal with AI
- Check Analysis Result
- Notify Not Food Image

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` (@n8n/n8n-nodes-langchain.lmChatGoogleGemini) — provides the language model connection for the chain node.
- **Key configuration:**
  - Uses **Google Gemini (PaLM) API** credentials (`googlePalmApi`).
  - Default model/options not explicitly set in parameters (depends on credential/node defaults).
- **Inputs/Outputs:**
  - **Output (ai_languageModel) →** Analyze Meal with AI (language model input)
- **Failure/edge cases:**
  - Credential misconfiguration (API key invalid, billing, region).
  - Model may refuse or produce non-conforming output; parser relies on exact “Key: Value” lines.
- **Version notes:** typeVersion `1`.

#### Node: Analyze Meal with AI
- **Type / role:** `chainLlm` (@n8n/n8n-nodes-langchain.chainLlm) — sends a prompt + image to the model and returns text.
- **Key configuration:**
  - **Prompt:** nutritionist role, strict formatting rules, exact keys required.
  - **Prompt expects image input:** `HumanMessagePromptTemplate` with `messageType: imageBinary`
  - The binary comes from **Download Image from LINE** (same execution branch).
- **Inputs/Outputs:**
  - **Main input ←** Download Image from LINE (binary)
  - **AI languageModel input ←** Google Gemini Chat Model
  - **Output →** Check Analysis Result
- **Failure/edge cases:**
  - If binary property name is not what the node expects (varies by node), the image may not be passed correctly.
  - Gemini may output localized punctuation (fullwidth colon `：`)—the parser partially normalizes this.
  - Output may include extra lines or formatting; parser ignores lines that don’t match `Key: Value`.
- **Version notes:** typeVersion `1.7`.

#### Node: Check Analysis Result
- **Type / role:** `If` (n8n-nodes-base.if) — filters out “Not a food image” responses.
- **Key configuration:**
  - Condition: `={{ $json.text }}` **notContains** `"Not a food image"`
- **Inputs/Outputs:**
  - **Input ←** Analyze Meal with AI
  - **True →** Parse Nutrition Data
  - **False →** Notify Not Food Image
- **Failure/edge cases:**
  - If Gemini returns “Not a food image” with different casing/wording, this gate may not catch it.
  - If output text is missing (`$json.text` undefined), condition may behave unexpectedly.
- **Version notes:** typeVersion `2.2`.

#### Node: Notify Not Food Image
- **Type / role:** `HTTP Request` — pushes a text message to the user via LINE.
- **Key configuration:**
  - POST `https://api.line.me/v2/bot/message/push`
  - JSON body includes:
    - `to`: `{{ $('LINE Webhook').item.json.body.events[0].source.userId }}`
    - Message text: asks user to send a clear meal photo
  - Headers:
    - `Content-Type: application/json`
    - `Authorization: Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Inputs/Outputs:**
  - **Input ←** Check Analysis Result (false branch)
  - No downstream nodes.
- **Failure/edge cases:**
  - Push messaging may require LINE plan/permissions; can fail even with valid token.
  - If webhook event comes from a group/room, `source.userId` may not be present or may differ; push may fail.
- **Version notes:** typeVersion `4.3`.

---

### Block 4 — Parsing, Merging, Storage, Logging
**Overview:** Converts the AI text to structured nutrition fields, merges those fields with the original image binary, uploads image to Drive, and appends a row to Sheets.

**Nodes involved:**
- Parse Nutrition Data
- Merge Data
- Upload to Google Drive
- Save to Google Sheets

#### Node: Parse Nutrition Data
- **Type / role:** `Code` (n8n-nodes-base.code) — robust-ish parser + derived fields.
- **Key configuration (interpreted behavior):**
  - Reads AI result text from `$json.text` (or fallback `$input.first().json.text`).
  - Normalizes:
    - trims lines
    - converts fullwidth colon `：` to `:`
    - splits into lines
  - Maps expected keys (case-insensitive) to internal field names:
    - `Food Items` → `foodItems`, `Estimated Calories` → `calories`, etc.
  - Numeric parsing: extracts first number (supports decimals) from value.
  - Adds:
    - `date` and `time` from `$now`
    - `dailyGoal` from Config (default 2000)
    - `remainingCalories = dailyGoal - calories`
    - `healthEmoji` and `healthLevel` based on `healthScore`
  - Returns JSON **and** preserves binary from `$input.first().binary` for later upload.
- **Inputs/Outputs:**
  - **Input ←** Check Analysis Result (true branch)
  - **Output →** Merge Data (to input index 1)
- **Failure/edge cases:**
  - If Gemini deviates from “Key: Value” format, fields may stay empty/0.
  - Negative `remainingCalories` possible if meal calories exceed goal.
  - Emoji output in this node conflicts with a “no emoji” policy only if your environment restricts it; technically it’s fine in n8n, but note your final LINE text includes emojis too.
  - `$now.toFormat(...)` depends on n8n Luxon availability (standard in n8n).
- **Version notes:** typeVersion `2`.

#### Node: Merge Data
- **Type / role:** `Merge` (n8n-nodes-base.merge) — combines image binary (from download) with parsed nutrition JSON (from code node).
- **Key configuration:**
  - **Mode:** combine
  - **Combine by:** position
  - Input 0: from Download Image from LINE
  - Input 1: from Parse Nutrition Data
- **Inputs/Outputs:**
  - **Input 0 ←** Download Image from LINE
  - **Input 1 ←** Parse Nutrition Data
  - **Output →** Upload to Google Drive
- **Failure/edge cases:**
  - If either branch produces 0 items or mismatched item counts, “combine by position” can drop or mis-pair data.
  - Binary property name must remain present after merge for Google Drive upload.
- **Version notes:** typeVersion `3`.

#### Node: Upload to Google Drive
- **Type / role:** `Google Drive` (n8n-nodes-base.googleDrive) — uploads the meal image.
- **Key configuration:**
  - File name: `Meal_yyyyMMdd_HHmmss.jpg` (from `$now`)
  - Target folder ID: `={{ $('Config').item.json.GOOGLE_DRIVE_FOLDER_ID }}`
  - Drive: “My Drive”
  - Uses OAuth2 credential: `googleDriveOAuth2Api`
- **Inputs/Outputs:**
  - **Input ←** Merge Data (should include binary)
  - **Output →** Save to Google Sheets
- **Failure/edge cases:**
  - Wrong folder ID or insufficient permissions → 403/404.
  - If the incoming binary isn’t attached under the expected binary key, upload fails.
  - The workflow later uses `webViewLink`; ensure the Drive node returns it (may depend on node operation/permissions).
- **Version notes:** typeVersion `3`.

#### Node: Save to Google Sheets
- **Type / role:** `Google Sheets` (n8n-nodes-base.googleSheets) — appends a new log row.
- **Key configuration:**
  - **Operation:** append
  - **DocumentId:** `={{ $('Config').item.json.GOOGLE_SHEETS_ID }}`
  - **Sheet:** `gid=0` (first sheet)
  - **Columns mapped:**
    - Date, Time, Meal Type, Food Items, Calories, Protein (g), Carbs (g), Fat (g), Fiber (g), Health Score, Advice
    - Image URL: `={{ $('Upload to Google Drive').item.json.webViewLink }}`
  - Uses OAuth2 credential: `googleSheetsOAuth2Api`
- **Inputs/Outputs:**
  - **Input ←** Upload to Google Drive
  - **Output →** Reply to LINE
- **Failure/edge cases:**
  - Sheet headers must match exactly (case/spacing) to avoid mis-mapping.
  - If the spreadsheet is not shared with the OAuth account, append will fail.
  - If `webViewLink` is missing, “Image URL” cell will be blank or expression error.
- **Version notes:** typeVersion `4.5`.

---

### Block 5 — LINE Response (Success Path)
**Overview:** Sends the final formatted nutrition analysis and remaining calories to the user via LINE Push message.

**Nodes involved:**
- Reply to LINE

#### Node: Reply to LINE
- **Type / role:** `HTTP Request` — push message to LINE user with formatted results.
- **Key configuration:**
  - POST `https://api.line.me/v2/bot/message/push`
  - Uses `to: {{ ...source.userId }}` from the webhook event.
  - Text template references many parsed fields:
    - mealType, foodItems, calories, protein, carbs, fat, fiber
    - healthEmoji, healthScore, healthLevel
    - positivePoints, improvementTips, advice
    - dailyGoal, remainingCalories
  - Headers:
    - `Content-Type: application/json`
    - `Authorization: Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Inputs/Outputs:**
  - **Input ←** Save to Google Sheets
  - No downstream nodes.
- **Failure/edge cases:**
  - Same LINE push constraints as above (permissions, userId availability).
  - If any referenced field is empty due to parsing issues, the message may look broken but still sends.
- **Version notes:** typeVersion `4.3`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview | — | — | ## 🍽️ AI Meal Nutrition Tracker / (contains how-it-works + setup steps + estimate disclaimer) |
| Step 1 - Trigger & Config | Sticky Note | Documentation for block 1 | — | — | ## Trigger & Configuration (LINE webhook + Config) |
| Step 2 - Image Processing | Sticky Note | Documentation for block 2/3 | — | — | ## AI Food Analysis (download + Gemini) |
| Step 3 - Data Storage | Sticky Note | Documentation for block 4 | — | — | ## Data Processing & Storage (parse + Drive + Sheets) |
| Step 4 - Response | Sticky Note | Documentation for block 5 | — | — | ## LINE Response (send formatted analysis) |
| LINE Webhook | Webhook | Entry point from LINE | — | Config | ## Trigger & Configuration (LINE webhook + Config) |
| Config | Set | Store tokens/IDs and constants | LINE Webhook | Check If Image | ## Trigger & Configuration (LINE webhook + Config) |
| Check If Image | Switch | Validate message.type == image | Config | Download Image from LINE | ## AI Food Analysis (download + Gemini) |
| Download Image from LINE | HTTP Request | Download image binary from LINE | Check If Image | Merge Data; Analyze Meal with AI | ## AI Food Analysis (download + Gemini) |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | LLM provider connection | — | Analyze Meal with AI (ai_languageModel) | ## AI Food Analysis (download + Gemini) |
| Analyze Meal with AI | LLM Chain (LangChain) | Prompt + image → nutrition text | Download Image from LINE; Google Gemini Chat Model | Check Analysis Result | ## AI Food Analysis (download + Gemini) |
| Check Analysis Result | If | Route based on “Not a food image” | Analyze Meal with AI | Parse Nutrition Data; Notify Not Food Image | ## AI Food Analysis (download + Gemini) |
| Notify Not Food Image | HTTP Request | Push fallback LINE message | Check Analysis Result (false) | — | ## LINE Response (send formatted analysis) |
| Parse Nutrition Data | Code | Parse key/value lines into structured fields | Check Analysis Result (true) | Merge Data | ## Data Processing & Storage (parse + Drive + Sheets) |
| Merge Data | Merge | Combine parsed JSON + image binary | Download Image from LINE; Parse Nutrition Data | Upload to Google Drive | ## Data Processing & Storage (parse + Drive + Sheets) |
| Upload to Google Drive | Google Drive | Upload image and return link | Merge Data | Save to Google Sheets | ## Data Processing & Storage (parse + Drive + Sheets) |
| Save to Google Sheets | Google Sheets | Append meal log row | Upload to Google Drive | Reply to LINE | ## Data Processing & Storage (parse + Drive + Sheets) |
| Reply to LINE | HTTP Request | Push formatted results to LINE | Save to Google Sheets | — | ## LINE Response (send formatted analysis) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *AI Meal Nutrition Tracker with LINE and Google Sheets* (or your preferred name).

2. **Add Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: **/MealTracker**
   - Copy the generated **Test/Production URL** (you will register it in LINE Developers).

3. **Add Config node**
   - Add node: **Set** (name it **Config**)
   - Add fields:
     - `LINE_CHANNEL_ACCESS_TOKEN` (String)
     - `GOOGLE_SHEETS_ID` (String)
     - `GOOGLE_DRIVE_FOLDER_ID` (String)
     - `DAILY_CALORIE_GOAL` (Number, e.g., 2000)
   - Connect: **Webhook → Config**

4. **Add Switch “Check If Image”**
   - Add node: **Switch**
   - Create rule: *String equals*
     - Value 1 (expression): `$('LINE Webhook').item.json.body.events[0].message.type`
     - Value 2: `image`
   - Connect: **Config → Check If Image**
   - Connect rule “true/match” output to next step (download).

5. **Add HTTP Request “Download Image from LINE”**
   - Add node: **HTTP Request**
   - Method: **GET** (default)
   - URL (expression):  
     `https://api-data.line.me/v2/bot/message/{{ $('LINE Webhook').item.json.body.events[0].message.id }}/content`
   - Response: **File** / “Response Format: File”
   - Send headers: enabled
   - Header:
     - `Authorization` = `Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
   - Connect: **Check If Image → Download Image from LINE**

6. **Add Google Gemini Chat Model**
   - Add node: **Google Gemini Chat Model** (LangChain node)
   - Create/attach credentials:
     - Credential type: **Google Gemini(PaLM) API** (`googlePalmApi`)
     - Provide API key (and any required project/billing setup)
   - Leave default model/options unless you need to pin a specific model.

7. **Add “Analyze Meal with AI” chain node**
   - Add node: **LLM Chain** (LangChain `chainLlm`)
   - Configure prompt text exactly with required keys (Food Items, Meal Type, etc.) and the sentinel string “Not a food image”.
   - Configure the message input to accept **imageBinary** (human message template with image binary).
   - Connect:
     - **Download Image from LINE (main)** → **Analyze Meal with AI (main)**
     - **Google Gemini Chat Model (ai_languageModel)** → **Analyze Meal with AI (ai_languageModel)**

8. **Add IF “Check Analysis Result”**
   - Add node: **If**
   - Condition:
     - String **does not contain**
     - Value (expression): `{{$json.text}}`
     - Search: `Not a food image`
   - Connect: **Analyze Meal with AI → Check Analysis Result**

9. **Add HTTP Request “Notify Not Food Image” (false branch)**
   - Add node: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.line.me/v2/bot/message/push`
   - Body type: **JSON**
   - JSON body (use expressions):
     - `to`: `{{ $('LINE Webhook').item.json.body.events[0].source.userId }}`
     - `messages`: array with one `{ type: "text", text: "...not food..." }`
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
   - Connect: **Check Analysis Result (false)** → **Notify Not Food Image**

10. **Add Code node “Parse Nutrition Data” (true branch)**
   - Add node: **Code**
   - Paste logic to:
     - split `$json.text` into lines
     - map keys to fields
     - parse numbers
     - compute remaining calories and health level/emoji
     - return `{ json: out, binary: $input.first().binary }`
   - Connect: **Check Analysis Result (true)** → **Parse Nutrition Data**

11. **Add Merge node “Merge Data”**
   - Add node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - **Download Image from LINE** → **Merge Data (Input 1 / index 0)**
     - **Parse Nutrition Data** → **Merge Data (Input 2 / index 1)**

12. **Add Google Drive “Upload to Google Drive”**
   - Add node: **Google Drive**
   - Operation: upload (binary upload)
   - File name (expression): `Meal_{{$now.format('yyyyMMdd_HHmmss')}}.jpg`
   - Folder ID (expression): `{{ $('Config').item.json.GOOGLE_DRIVE_FOLDER_ID }}`
   - Credentials: **Google Drive OAuth2**
     - Ensure OAuth account has access to the target folder
   - Connect: **Merge Data → Upload to Google Drive**

13. **Add Google Sheets “Save to Google Sheets”**
   - Add node: **Google Sheets**
   - Operation: **Append**
   - Document ID (expression): `{{ $('Config').item.json.GOOGLE_SHEETS_ID }}`
   - Sheet: pick the first sheet (gid=0) or your specific tab
   - Map columns to expressions from **Parse Nutrition Data** and Drive link:
     - Date, Time, Meal Type, Food Items, Calories, Protein (g), Carbs (g), Fat (g), Fiber (g), Health Score, Advice
     - Image URL: `{{ $('Upload to Google Drive').item.json.webViewLink }}`
   - Credentials: **Google Sheets OAuth2**
   - Connect: **Upload to Google Drive → Save to Google Sheets**
   - Ensure your Google Sheet has headers matching the column names you map.

14. **Add HTTP Request “Reply to LINE”**
   - Add node: **HTTP Request**
   - POST `https://api.line.me/v2/bot/message/push`
   - JSON body with `to` userId and formatted `text` using fields from **Parse Nutrition Data**
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer {{ $('Config').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
   - Connect: **Save to Google Sheets → Reply to LINE**

15. **LINE platform setup**
   - In LINE Developers Console:
     - Set the webhook URL to your n8n webhook production URL ending with `/MealTracker`
     - Enable webhook
     - Ensure your bot has permission for push messages if you use `/push` endpoint.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Nutritional values are AI estimates and should not replace professional dietary advice. | From sticky note “Workflow Overview” |
| Setup prerequisites: LINE Messaging API channel + Channel Access Token; Google Sheets with columns (Date, Time, Meal Type, Food Items, Calories, Protein, Carbs, Fat, Fiber, Health Score, Advice); Google Drive folder; set webhook URL in LINE console. | From sticky note “Workflow Overview” |