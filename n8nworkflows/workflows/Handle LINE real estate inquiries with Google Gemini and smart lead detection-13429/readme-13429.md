Handle LINE real estate inquiries with Google Gemini and smart lead detection

https://n8nworkflows.xyz/workflows/handle-line-real-estate-inquiries-with-google-gemini-and-smart-lead-detection-13429


# Handle LINE real estate inquiries with Google Gemini and smart lead detection

## 1. Workflow Overview

**Purpose:**  
This workflow implements a LINE Messaging Bot “real estate assistant” that automatically parses incoming customer messages, generates a professional Japanese reply using **Google Gemini**, logs the interaction to **Google Sheets**, and triggers an **email alert** when a lead is considered high value.

**Target use cases:**
- Automating first-response handling for rental/purchase/viewing inquiries on LINE
- Capturing structured lead data (budget/rooms/area) from free-text Japanese messages
- Alerting a sales team when the message indicates purchase intent or high budget

### 1.1 Input Reception & Parsing
Receives LINE webhook events, extracts message/user data, classifies inquiry type, and pulls structured fields (budget/rooms/area).

### 1.2 AI Processing (Google Gemini)
Uses a Gemini chat model (Gemini 1.5 Flash) via an LLM chain node to produce a short, polite Japanese reply (≤200 chars).

### 1.3 Logging & Lead Detection
Enriches the payload with AI output and a boolean “high value” flag, appends the record to Google Sheets, and branches if high value.

### 1.4 Reply & Notifications
Sends the AI reply back to LINE via HTTP request; sends an email alert for high-value leads.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Parsing
**Overview:**  
Receives LINE webhook requests and converts the first event into structured lead fields used downstream (classification + budget/rooms/area extraction).

**Nodes involved:**  
- LINE Webhook  
- Parse LINE Message

#### Node: LINE Webhook
- **Type / Role:** `n8n-nodes-base.webhook` — entry point receiving POST calls from LINE.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `line-realestate-webhook`
  - **Webhook ID:** `line-realestate-webhook-001` (internal identifier in the workflow)
- **Inputs / Outputs:**
  - **Output →** Parse LINE Message
- **Version notes:** typeVersion `2.1`
- **Edge cases / failures:**
  - LINE signature validation is **not** implemented here (LINE usually sends `X-Line-Signature`). Without verification, spoofed requests are possible.
  - Payload structure differences (non-message events, multiple events) can break parsing assumptions downstream.

#### Node: Parse LINE Message
- **Type / Role:** `n8n-nodes-base.code` — parses webhook JSON and extracts structured fields.
- **Key configuration choices:**
  - Reads webhook body from: `$input.first().json.body`
  - Uses only the **first** event: `events[0]`
  - Extracts:
    - `userMessage` from `event.message?.text`
    - `userId` from `event.source?.userId`
    - `replyToken` from `event.replyToken`
  - Classifies inquiry type by keyword matching:
    - rental if includes `賃貸` or `家賃`
    - purchase if includes `売買` or `購入` or `買い`
    - viewing if includes `内見` or `見学`
    - else `general`
  - Extracts key info via regex:
    - **Budget:** `(\d+)万` → integer “万円” (e.g., `3500万` → `3500`)
    - **Rooms:** `(\d+)[LDK]` → captures patterns like `2LDK` (returns the match string)
    - **Area:** `([\u3040-\u309f\u4e00-\u9faf]+[区市町村])` → Japanese characters ending with 区/市/町/村
  - Adds `timestamp` as ISO string.
- **Inputs / Outputs:**
  - **Input ←** LINE Webhook
  - **Output →** AI Property Advisor
- **Version notes:** typeVersion `2`
- **Edge cases / failures:**
  - If `body.events` is missing/empty, returns `{ skip: true }` but the workflow still continues (no IF node checks `skip`), so later nodes may generate meaningless AI replies/logs.
  - Non-text messages (stickers, images) will set `userMessage` to `''`; classification and regex extraction will be empty.
  - Budget extraction only recognizes “万” format (e.g., “3,000万円”, “3000万円”, “3千万” won’t match reliably).
  - Area regex may over/under-match; does not capture prefectures or neighborhoods not ending in 区/市/町/村.

---

### Block 2 — AI Processing (Google Gemini)
**Overview:**  
Generates a polite Japanese real estate response using structured fields extracted from the LINE message and constraints suitable for LINE (short response).

**Nodes involved:**  
- AI Property Advisor  
- Google Gemini Chat Model

#### Node: AI Property Advisor
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — LLM chain node that sends a prompt to a connected language model.
- **Key configuration choices:**
  - **Prompt type:** “define” (custom prompt text)
  - Prompt includes:
    - `{{ $json.userMessage }}`
    - `{{ $json.inquiryType }}`
    - `{{ $json.budget ? $json.budget + '万円' : '未指定' }}`
    - `{{ $json.rooms || '未指定' }}`
    - `{{ $json.area || '未指定' }}`
  - Enforces response rules:
    - polite Japanese (敬語), friendly
    - make concrete suggestions
    - ask naturally for missing info
    - encourage viewing/consultation
    - **≤ 200 characters** for LINE
    - includes a formatted structure with house/phone markers
- **Inputs / Outputs:**
  - **Input ←** Parse LINE Message
  - **Output →** Process AI Response
  - **Model input connection (AI) ←** Google Gemini Chat Model via `ai_languageModel`
- **Version notes:** typeVersion `1.4` (LangChain nodes change frequently across n8n versions; confirm compatibility if importing into older n8n)
- **Edge cases / failures:**
  - If the model returns longer than 200 characters, LINE may still accept it but UX/constraints may be violated (no hard truncation is applied).
  - If upstream payload is missing fields (e.g., skip case), the prompt still runs.
  - Model/credential misconfiguration will fail execution.

#### Node: Google Gemini Chat Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides Gemini chat model to the chain node.
- **Key configuration choices:**
  - **Model:** `models/gemini-1.5-flash`
  - **Temperature:** `0.3` (more consistent, less creative)
- **Inputs / Outputs:**
  - **Output (ai_languageModel) →** AI Property Advisor
- **Version notes:** typeVersion `1`
- **Edge cases / failures:**
  - Requires valid Gemini credentials and enabled API access.
  - Rate limits / quota exhaustion / region restrictions can interrupt replies.

---

### Block 3 — Logging & Lead Detection
**Overview:**  
Merges the AI response back into the structured payload, flags high-value leads, writes the interaction to Google Sheets, and branches for sales alerts.

**Nodes involved:**  
- Process AI Response  
- Log to Google Sheets  
- High-Value Lead Check

#### Node: Process AI Response
- **Type / Role:** `n8n-nodes-base.code` — normalizes AI output and computes lead scoring flag.
- **Key configuration choices:**
  - Reads AI response from:  
    - `$input.first().json.text` OR `$input.first().json.response` (fallback)
  - Pulls original parsed data using a **node reference**:  
    - `$('Parse LINE Message').first().json`
  - Computes `isHighValue`:
    - true if `inquiryType === 'purchase'`
    - OR if `budget > 3000` (i.e., > 3000万円 = > ¥30M)
  - Returns combined JSON: original fields + `aiResponse` + `isHighValue`
- **Inputs / Outputs:**
  - **Input ←** AI Property Advisor
  - **Outputs →**
    - Log to Google Sheets
    - High-Value Lead Check
- **Version notes:** typeVersion `2`
- **Edge cases / failures:**
  - If AI Property Advisor returns output in a different property than `text`/`response`, `aiResponse` becomes empty.
  - If Parse LINE Message produced `{skip:true}` (no events), this node will still compute and propagate mostly empty lead data.
  - Budget comparison assumes `budget` is numeric in “万円” units.

#### Node: Log to Google Sheets
- **Type / Role:** `n8n-nodes-base.googleSheets` — appends a row to a spreadsheet as an interaction log.
- **Key configuration choices:**
  - **Operation:** Append
  - **Document ID:** not set in the template (`value` is empty)
  - **Sheet name:** not set in the template (`value` is empty)
  - In practice, you must map columns (not shown here) or rely on default behavior depending on n8n UI setup.
- **Inputs / Outputs:**
  - **Input ←** Process AI Response
  - **Output →** Send LINE Reply
- **Version notes:** typeVersion `4.5`
- **Edge cases / failures:**
  - Missing document/sheet configuration will cause runtime failure.
  - Google auth issues (OAuth refresh failures) and permissions (sheet not shared) are common.
  - Column mismatch: if the sheet expects specific headers, append may fail or place data incorrectly unless field mapping is configured.

#### Node: High-Value Lead Check
- **Type / Role:** `n8n-nodes-base.if` — branches based on `isHighValue`.
- **Key configuration choices:**
  - Condition: `{{ $json.isHighValue }}` equals `true` (boolean strict validation enabled)
- **Inputs / Outputs:**
  - **Input ←** Process AI Response
  - **True output →** Send Sales Alert
  - **False output →** (not connected)
- **Version notes:** typeVersion `2`
- **Edge cases / failures:**
  - If `isHighValue` is undefined (bad upstream), strict type validation can fail comparisons or evaluate to false depending on n8n behavior/version.

---

### Block 4 — Reply & Notifications
**Overview:**  
Sends the generated response back to the user on LINE, and emails the sales team if the inquiry is high value.

**Nodes involved:**  
- Send LINE Reply  
- Send Sales Alert

#### Node: Send LINE Reply
- **Type / Role:** `n8n-nodes-base.httpRequest` — calls LINE Reply API.
- **Key configuration choices:**
  - **URL:** `https://api.line.me/v2/bot/message/reply`
  - **Method:** POST
  - **Body:** JSON (constructed with expression)
    - `replyToken` from `{{ $json.replyToken }}`
    - message text from `{{ $json.aiResponse }}`
  - **Headers:**
    - `Content-Type: application/json`
    - `Authorization: Bearer YOUR_LINE_CHANNEL_ACCESS_TOKEN` (placeholder; should be replaced with a credential or env var)
- **Inputs / Outputs:**
  - **Input ←** Log to Google Sheets
  - **Output →** none
- **Version notes:** typeVersion `4.2`
- **Edge cases / failures:**
  - Using a hardcoded token is fragile; token rotation breaks the workflow.
  - LINE reply tokens expire quickly and can be used only once; delays (e.g., slow AI, Sheets latency) can cause LINE API errors.
  - If `aiResponse` is empty, LINE will still receive an empty/invalid message; API may reject.

#### Node: Send Sales Alert
- **Type / Role:** `n8n-nodes-base.emailSend` — sends an email notification for premium leads.
- **Key configuration choices:**
  - **To:** `user@example.com` (placeholder)
  - **From:** `user@example.com` (placeholder)
  - **Subject:** `🏠 高額案件アラート: {{ $json.inquiryType }} - {{ $json.area || 'エリア未指定' }}`
  - SMTP credentials must be configured in n8n for this node to send.
- **Inputs / Outputs:**
  - **Input ←** High-Value Lead Check (true branch only)
  - **Output →** none
- **Version notes:** typeVersion `2.1`
- **Edge cases / failures:**
  - Missing SMTP credentials or provider blocks (SPF/DKIM, “from” restrictions) will fail.
  - No email body is configured here; depending on n8n defaults, the email may be empty except subject (often undesirable).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview |  |  | ## 🏠 LINE AI Real Estate Assistant … ⚠️ Note: Requires LINE Messaging API, Google Gemini, Google Sheets, and SMTP credentials. |
| Step 1 - Reception & Parsing | Sticky Note | Block label / guidance |  |  | ### Step 1 - Message Reception & Parsing … **Setup:** LINE Channel Access Token + Webhook URL |
| LINE Webhook | Webhook | Receives LINE events (entry) |  | Parse LINE Message | ### Step 1 - Message Reception & Parsing … **Setup:** LINE Channel Access Token + Webhook URL |
| Parse LINE Message | Code | Extracts fields + classifies inquiry | LINE Webhook | AI Property Advisor | ### Step 1 - Message Reception & Parsing … **Setup:** LINE Channel Access Token + Webhook URL |
| Step 2 - AI Analysis | Sticky Note | Block label / guidance |  |  | ### Step 2 - AI Property Advisor … **Model:** Gemini 1.5 Flash **Language:** Japanese (敬語) **Limit:** 200 chars for LINE |
| AI Property Advisor | LangChain Chain LLM | Prompt orchestration for AI reply | Parse LINE Message (+ Gemini model via AI input) | Process AI Response | ### Step 2 - AI Property Advisor … **Model:** Gemini 1.5 Flash **Language:** Japanese (敬語) **Limit:** 200 chars for LINE |
| Google Gemini Chat Model | LangChain Chat Model (Gemini) | Provides Gemini model to the chain |  | AI Property Advisor (ai_languageModel) | ### Step 2 - AI Property Advisor … **Model:** Gemini 1.5 Flash **Language:** Japanese (敬語) **Limit:** 200 chars for LINE |
| Step 3 - Logging & Routing | Sticky Note | Block label / guidance |  |  | ### Step 3 - Logging & Lead Detection … **Filter:** Purchase OR budget >3000万円 |
| Process AI Response | Code | Merge AI output + compute high-value flag | AI Property Advisor | Log to Google Sheets; High-Value Lead Check | ### Step 3 - Logging & Lead Detection … **Filter:** Purchase OR budget >3000万円 |
| Log to Google Sheets | Google Sheets | Append interaction record | Process AI Response | Send LINE Reply | ### Step 3 - Logging & Lead Detection … **Filter:** Purchase OR budget >3000万円 |
| High-Value Lead Check | IF | Route high-value leads to email | Process AI Response | Send Sales Alert (true) | ### Step 3 - Logging & Lead Detection … **Filter:** Purchase OR budget >3000万円 |
| Step 4 - Reply & Alerts | Sticky Note | Block label / guidance |  |  | ### Step 4 - Reply & Notifications … **LINE Reply:** Auto-response with AI advice **Email Alert:** High-value lead notification |
| Send LINE Reply | HTTP Request | Calls LINE Reply API | Log to Google Sheets |  | ### Step 4 - Reply & Notifications … **LINE Reply:** Auto-response with AI advice **Email Alert:** High-value lead notification |
| Send Sales Alert | Email Send | Sends premium lead email | High-Value Lead Check (true) |  | ### Step 4 - Reply & Notifications … **LINE Reply:** Auto-response with AI advice **Email Alert:** High-value lead notification |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Sticky Notes (optional but matches structure)**
   - Add four sticky notes named:
     - “Workflow Overview”
     - “Step 1 - Reception & Parsing”
     - “Step 2 - AI Analysis”
     - “Step 3 - Logging & Routing”
     - “Step 4 - Reply & Alerts”
   - Paste the corresponding text content from the workflow for operational context.

2) **Add the entry node: “LINE Webhook”**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `line-realestate-webhook`
   - Save the workflow to generate a production webhook URL.
   - In LINE Developers Console, set the bot webhook URL to the n8n webhook URL for this node.

3) **Add “Parse LINE Message” (Code node)**
   - Node type: **Code**
   - Paste the JavaScript that:
     - reads `$input.first().json.body.events`
     - extracts `userId`, `replyToken`, `userMessage`
     - classifies into `inquiryType`
     - extracts `budget` (万円 integer), `rooms`, `area`
     - sets `timestamp`
   - Connect: **LINE Webhook → Parse LINE Message**

4) **Add Gemini model node: “Google Gemini Chat Model”**
   - Node type: **Google Gemini Chat Model** (LangChain)
   - Model: `models/gemini-1.5-flash`
   - Temperature: `0.3`
   - **Credentials:** configure Google Gemini / Google AI Studio credential in n8n (API key or supported auth method depending on your n8n version).

5) **Add LLM chain node: “AI Property Advisor”**
   - Node type: **Chain LLM**
   - Prompt type: **Define**
   - Paste the Japanese prompt template including these expressions:
     - `{{ $json.userMessage }}`
     - `{{ $json.inquiryType }}`
     - `{{ $json.budget ? $json.budget + '万円' : '未指定' }}`
     - `{{ $json.rooms || '未指定' }}`
     - `{{ $json.area || '未指定' }}`
   - Connect main flow: **Parse LINE Message → AI Property Advisor**
   - Connect AI model input: **Google Gemini Chat Model (ai_languageModel) → AI Property Advisor**

6) **Add “Process AI Response” (Code node)**
   - Node type: **Code**
   - Implement logic to:
     - read AI output from `$input.first().json.text || $input.first().json.response || ''`
     - read original parsed data via `$('Parse LINE Message').first().json`
     - compute `isHighValue = inquiryType === 'purchase' || (budget && budget > 3000)`
     - output combined JSON including `aiResponse` and `isHighValue`
   - Connect: **AI Property Advisor → Process AI Response**

7) **Add “Log to Google Sheets”**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Configure:
     - **Document ID:** select your spreadsheet
     - **Sheet name:** select the logging sheet/tab
     - Map columns to fields such as: timestamp, userId, userMessage, inquiryType, budget, rooms, area, aiResponse, isHighValue
   - **Credentials:** Google Sheets OAuth2 credential with access to the target sheet.
   - Connect: **Process AI Response → Log to Google Sheets**

8) **Add “High-Value Lead Check” (IF node)**
   - Node type: **IF**
   - Condition: boolean equals
     - Left value: `{{ $json.isHighValue }}`
     - Right value: `true`
   - Connect: **Process AI Response → High-Value Lead Check**

9) **Add “Send LINE Reply” (HTTP Request node)**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Send body: **JSON**
   - Body (expression-driven):
     - `replyToken`: `{{ $json.replyToken }}`
     - `messages[0].text`: `{{ $json.aiResponse }}`
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer <LINE_CHANNEL_ACCESS_TOKEN>`
   - Best practice: store token in n8n credentials or environment variable rather than hardcoding.
   - Connect: **Log to Google Sheets → Send LINE Reply**

10) **Add “Send Sales Alert” (Email Send node)**
   - Node type: **Email Send**
   - Configure SMTP credentials in n8n (host/port/user/pass or provider-supported method).
   - Set:
     - To: sales team address(es)
     - From: allowed sender address
     - Subject expression: `🏠 高額案件アラート: {{ $json.inquiryType }} - {{ $json.area || 'エリア未指定' }}`
     - Add a body (recommended) including userMessage, budget, userId, timestamp, aiResponse.
   - Connect: **High-Value Lead Check (true) → Send Sales Alert**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Requires LINE Messaging API, Google Gemini, Google Sheets, and SMTP credentials.” | From sticky note “Workflow Overview” |
| “Set webhook URL in LINE Developer Console” | From sticky note “Workflow Overview” |
| High-value lead rule: Purchase OR budget > 3000万円 | From sticky note “Step 3 - Logging & Routing” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.