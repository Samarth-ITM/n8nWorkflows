Real-time lead response across social channels with Llama AI & Google Sheets

https://n8nworkflows.xyz/workflows/real-time-lead-response-across-social-channels-with-llama-ai---google-sheets-12094


# Real-time lead response across social channels with Llama AI & Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow acts as an “instant SDR worker” that receives inbound lead messages from multiple social/communication channels (WhatsApp, Instagram, Facebook, LinkedIn, Website), generates a short context-aware reply using **Llama (via Groq Chat Completions API)**, sends the reply back on the originating channel, then logs the interaction into **Google Sheets**.

**Target use cases:**
- Real-time lead response automation across social inboxes and website inquiries
- Consistent first-touch replies with working-hours awareness (IST)
- Centralized lead + response logging for sales follow-up

### 1.1 Lead Intake (Multi-Webhook Entry Points)
Multiple Webhook triggers receive channel-specific payloads.

### 1.2 Data Normalization (Standard Lead Schema)
Each channel payload is mapped into a shared structure under `body.*`.

### 1.3 Channel Classification (Pre-processing Switch)
A Switch node routes all normalized items forward (currently all cases route to the same next step, acting mainly as a validation/branch placeholder).

### 1.4 IST Working Hours Evaluation
A Code node converts submission time to IST and derives day/time/working-hours booleans, then an IF node checks working-day + working-hours.

### 1.5 AI Request Construction → Groq Llama Completion
A Code node builds an OpenAI-compatible request body for Groq, an HTTP Request node calls Groq, then a Code node extracts the model response and merges it back into a compact object.

### 1.6 Reply Routing & Sending (Per Channel)
A Switch routes to the appropriate “send” node for the source channel (WhatsApp / Instagram / Facebook / LinkedIn / email via Gmail).

### 1.7 Status Consolidation & Logging
A Code node determines send status and shapes a final log record, then Google Sheets append-or-update writes/updates the row (matching on Email).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Lead Intake & Normalization
**Overview:** Receives incoming lead messages via channel-specific webhooks and maps them into a unified schema at `body.*` (name, phone/email, message, channel, timestamp).  
**Nodes involved:**  
- Incomming Lead whatsapp1  
- Incomming Lead facebook1  
- Incomming Lead instagram1  
- Incomming Lead linkdin1  
- Incomming Lead Website1  
- Normalize Lead Data6 (WhatsApp)  
- Normalize Lead Data7 (Facebook)  
- Normalize Lead Data8 (Instagram)  
- Normalize Lead Data9 (LinkedIn)  
- Normalize Lead Data5 (Website)

#### Node: Incomming Lead whatsapp1
- **Type/role:** Webhook trigger (`POST /incoming-leads-whatsapp`)
- **Config:** HTTP Method POST; path `incoming-leads-whatsapp`
- **Output:** Emits incoming request body in `$json.body`
- **Failure modes:** not receiving requests (URL mismatch), missing fields expected by downstream Set node, webhook not activated/production URL not used

#### Node: Incomming Lead facebook1
- **Type/role:** Webhook trigger (`POST /incoming-leads-facebook`)
- **Config:** path `incoming-leads-facebook`
- **Failure modes:** same as above

#### Node: Incomming Lead instagram1
- **Type/role:** Webhook trigger (`POST /incoming-leads-instagram`)
- **Config:** path `incoming-leads-instagram`
- **Failure modes:** same as above

#### Node: Incomming Lead linkdin1
- **Type/role:** Webhook trigger (`POST /incoming-leads-linkdin`)
- **Config:** path `incoming-leads-linkdin`
- **Failure modes:** same as above (also note “linkdin” spelling appears throughout—keep consistent or fix everywhere)

#### Node: Incomming Lead Website1
- **Type/role:** Webhook trigger (`POST /incoming-leads-website`)
- **Config:** path `incoming-leads-website`
- **Failure modes:** same as above

#### Node: Normalize Lead Data6 (WhatsApp)
- **Type/role:** Set node to normalize payload
- **Key mappings:**
  - `body.lead_name = $json.body.ProfileName`
  - `body.contact_phone = $json.body.From`
  - `body.message_text = $json.body.Body`
  - `body.source_channel = "whatsapp"`
  - `body.submitted_at = new Date().toISOString()`
- **Input:** Webhook item
- **Output:** Standardized `body.*`
- **Edge cases:** upstream payload keys may differ (ProfileName/From/Body); missing values propagate to AI + sending step (e.g., WhatsApp recipient number)

#### Node: Normalize Lead Data7 (Facebook)
- Same mapping as WhatsApp but `body.source_channel = "facebook"`
- **Edge case:** later Switches look for `"Facebook"` (capital F) in one place and `"Facebook"` again—however the normalization sets `"facebook"` (lowercase). This mismatch can break routing.

#### Node: Normalize Lead Data8 (Instagram)
- Same mapping but `body.source_channel = "instagram"`
- **Edge case:** later Switches check `"Instagram"` (capital I). Mismatch can break routing.

#### Node: Normalize Lead Data9 (LinkedIn)
- Same mapping but `body.source_channel = "linkdin"`
- **Edge case:** later Switches check `"LinkedIn"` (proper spelling) in one place; mismatch can break routing.

#### Node: Normalize Lead Data5 (Website)
- **Type/role:** Set node normalization for website payload (already uses `body.*`)
- **Key mappings:**
  - `body.lead_name = $json.body.lead_name`
  - `body.contact_phone = $json.body.contact_phone`
  - `body.contact_email = $json.body.contact_email`
  - `body.message_text = $json.body.message_text`
  - `body.source_channel = $json.body.source_channel`
  - `body.submitted_at = new Date().toISOString()`
- **Edge cases:** if website form uses different field names; if `source_channel` isn’t “website” exactly, later routing may fail

---

### Block 2.2 — Channel Classification (Switch2)
**Overview:** Routes by `body.source_channel` into branches, but all branches currently converge to the same next node. This effectively acts as a validation/placeholder for future per-channel preprocessing.  
**Nodes involved:** Switch2

#### Node: Switch2
- **Type/role:** Switch (multi-branch router)
- **Rules:** compares `={{ $json.body.source_channel }}` to:
  - `whatsapp`
  - `Instagram`
  - `Facebook`
  - `LinkedIn`
  - `website`
- **Output:** Every rule output connects to **Extract Day and Hours1**
- **Edge cases / failure:**
  - **Case sensitivity mismatch**: normalization sets `instagram/facebook/linkdin` lowercased or misspelled, but switch uses capitalized “Instagram/Facebook/LinkedIn”.
  - If no rule matches, item will be dropped unless a “fallback” output is configured (not shown here).

---

### Block 2.3 — IST Working Hours Computation & Check
**Overview:** Converts submitted time to IST, computes day/time and a boolean flag indicating whether it’s within working hours (Mon–Fri, 09:30–18:30 IST), then gates downstream logic through an IF node (both branches currently converge).  
**Nodes involved:** Extract Day and Hours1, Is Working Day and Working Hour?1

#### Node: Extract Day and Hours1
- **Type/role:** Code node (JavaScript) for time conversion and business-hours evaluation
- **Key logic:**
  - Parses `new Date($json.body.submitted_at.trim())`
  - Adds IST offset (+330 minutes)
  - Derives:
    - `ist_dayOfWeek` (0–6)
    - `ist_timeHHMM` (HH:MM)
    - `ist_totalMinutes`
    - `isWorkingDay` (Mon–Fri)
    - `is_working_hours` (09:30–18:30 IST)
- **Output:** Merges computed fields back into original JSON
- **Failure modes:** invalid/missing `body.submitted_at` (Date parsing yields “Invalid Date”); timezone assumptions if `submitted_at` not UTC/ISO

#### Node: Is Working Day and Working Hour?1
- **Type/role:** IF node to check working day/hours
- **Conditions used:**
  - `ist_dayOfWeek < 6`
  - `isWorkingDay is true`
  - `is_working_hours is true`
- **Connections:** Both **true** and **false** outputs go to **Code in JavaScript3**
- **Practical effect:** Currently does not change path; it exists mainly to provide the boolean and for future branching.
- **Edge cases:** condition includes redundant `ist_dayOfWeek < 6` since `isWorkingDay` already excludes weekend; if booleans missing, strict validation may fail.

---

### Block 2.4 — AI Response Generation (Groq + Llama)
**Overview:** Builds a chat completion request instructing the model to respond differently for website (email style) vs social chats, and to mention office hours when outside working hours. Calls Groq’s OpenAI-compatible endpoint and extracts the generated response text.  
**Nodes involved:** Code in JavaScript3, Get Ai Response1, Code in JavaScript4

#### Node: Code in JavaScript3
- **Type/role:** Code node building the request payload for Groq
- **Model:** `llama-3.3-70b-versatile`
- **Prompt strategy:**
  - System: “professional SDR assistant… short, friendly, clear… do not mention AI”
  - User includes:
    - `SOURCE CHANNEL`, `WORKING HOURS`, lead name, lead message
    - Rules for website vs others; mention office hours if `WORKING HOURS` is false
- **Output:** Adds `requestBody` to the item JSON
- **Edge cases:** if any referenced fields are missing (e.g., `body.message_text`), output still builds but content may be low quality

#### Node: Get Ai Response1
- **Type/role:** HTTP Request to Groq Chat Completions (OpenAI compatible)
- **Endpoint:** `POST https://api.groq.com/openai/v1/chat/completions`
- **Headers:**
  - `Authorization: your-authorization-token` (should be `Bearer <token>` in most Groq setups)
  - `Content-Type: application/json`
- **Body:** JSON `={{ $json.requestBody }}`
- **Failure modes:**
  - 401/403 due to invalid token/header format
  - 429 rate limits
  - timeouts/network issues
  - schema mismatch if Groq expects different model name or payload fields

#### Node: Code in JavaScript4
- **Type/role:** Code node to extract response and merge with original lead data
- **Key expressions:**
  - `const aiResponse = $input.item.json.choices[0].message.content;`
  - Reads original from `$('Code in JavaScript3').item.json;`
- **Output JSON shape:**
  - `body` (normalized lead)
  - `ai_response`
  - working-hour and IST fields
- **Edge cases:**
  - If Groq error response does not include `choices[0]...`, this will throw
  - Using `$('Code in JavaScript3').item.json` assumes the node name is unchanged and execution has that item available (can break if renamed or multiple items are processed unexpectedly)

---

### Block 2.5 — Send Reply Routing & Delivery
**Overview:** Routes the AI response to the correct sending mechanism depending on `body.source_channel` and executes the send.  
**Nodes involved:** Switch3, Send message (WhatsApp), Send Instagram Message1, Send Facebook Messages1, Send Linkdin Messages1, Send a message1 (Gmail)

#### Node: Switch3
- **Type/role:** Switch router for sending step
- **Rules compare:** `={{ $json.body.source_channel }}` to:
  - `whatsapp`
  - `Instagram`
  - `Facebook`
  - `LinkedIn`
  - `website`
- **Outputs:**
  - whatsapp → Send message (WhatsApp)
  - Instagram → Send Instagram Message1
  - Facebook → Send Facebook Messages1
  - LinkedIn → Send Linkdin Messages1
  - website → Send a message1 (Gmail)
- **Critical edge case:** same **case/spelling mismatch** risks dropping items (e.g., normalized `instagram` won’t match `Instagram`).

#### Node: Send message (WhatsApp)
- **Type/role:** WhatsApp node (send message)
- **Config:**
  - Operation: `send`
  - `recipientPhoneNumber = {{$json.body.contact_phone}}`
  - `textBody = {{$json.ai_response}}`
  - `phoneNumberId`: placeholder
- **Failure modes:** invalid/unauthorized Meta WhatsApp credentials, wrong phone number format, blocked template/policy issues, missing `contact_phone`

#### Node: Send Instagram Message1
- **Type/role:** HTTP Request placeholder to Instagram API
- **Config:** URL `your-instagram-api-endpoint`
- **Missing details:** method/auth/body not specified—must be implemented for real sending
- **Failure modes:** auth errors, wrong endpoint, missing required payload fields

#### Node: Send Facebook Messages1
- **Type/role:** HTTP Request placeholder to Facebook API
- **Config:** URL `your-facebook-api-endpoint`
- **Failure modes:** same as Instagram

#### Node: Send Linkdin Messages1
- **Type/role:** HTTP Request placeholder to LinkedIn API
- **Config:** URL `your-linkdin-api-endpoint`
- **Failure modes:** same; also LinkedIn messaging APIs are restricted—may require partner access

#### Node: Send a message1 (Gmail)
- **Type/role:** Gmail node to send an email response for website leads
- **Config:**
  - To: `={{ $json.body.contact_email }}`
  - Subject: `Website developement Pricing Inquiry`
  - Message: `={{ $json.ai_response }}`
- **Failure modes:** OAuth credential expiration, missing `contact_email`, Gmail sending limits

---

### Block 2.6 — Status Consolidation & Google Sheets Logging
**Overview:** Consolidates send result, determines status, shapes a logging record including timestamps and working-hours context, and writes to Google Sheets using append-or-update keyed by Email.  
**Nodes involved:** Code in JavaScript5, google-sheet-name

#### Node: Code in JavaScript5
- **Type/role:** Code node creating a final “log row” JSON
- **Key logic:**
  - Reads current send response: `$input.item.json`
  - Pulls original data from `$('Code in JavaScript4').item.json`
  - Determines send status:
    - HTTP success if `statusCode` 2xx
    - Gmail success if `id` or `messageId` or `labelIds`
    - Failure if `error` exists
  - Outputs fields:
    - `timestamp`, `lead_name`, `email`, `phone`, `source_channel`, `lead_message`
    - `ai_response`
    - `working_hours`, `response_time_ist`, `day_of_week`
    - `status`, `channel_used`, `response_sent_at`
- **Edge cases:**
  - Some send nodes may not return `statusCode`; logic may mark processed but not “sent successfully”
  - Hard dependency on node name `Code in JavaScript4` (renaming breaks)
  - If sending fails hard (node error), this node may never run unless n8n error handling is configured

#### Node: google-sheet-name
- **Type/role:** Google Sheets append or update
- **Operation:** `appendOrUpdate`
- **Matching column:** `Email` (used as unique key)
- **Mapped columns:**
  - Lead Name, Email, Phone, Lead Message, Timestamp, Source Channel, Working Hours, Day of Week, Response Time, AI Response, Status
- **Config requirements:**
  - Document: `your-google-sheet-id`
  - Sheet: `gid=0` (selected by list)
- **Failure modes:**
  - OAuth/credentials issues
  - Sheet schema mismatch (column headers differ)
  - If Email is blank for non-website channels, appendOrUpdate matching can behave unexpectedly (may overwrite same blank key or fail)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Incomming Lead whatsapp1 | Webhook | Entry point for WhatsApp leads | — | Normalize Lead Data6 | ## Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)  |
| Incomming Lead facebook1 | Webhook | Entry point for Facebook leads | — | Normalize Lead Data7 | ## Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)  |
| Incomming Lead instagram1 | Webhook | Entry point for Instagram leads | — | Normalize Lead Data8 | ## Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)  |
| Incomming Lead linkdin1 | Webhook | Entry point for LinkedIn leads | — | Normalize Lead Data9 | ## Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)  |
| Incomming Lead Website1 | Webhook | Entry point for Website leads | — | Normalize Lead Data5 | ## Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)  |
| Normalize Lead Data6 | Set | Normalize WhatsApp payload into `body.*` | Incomming Lead whatsapp1 | Switch2 | ## Step 1: Lead Intake & Normalization |
| Normalize Lead Data7 | Set | Normalize Facebook payload into `body.*` | Incomming Lead facebook1 | Switch2 | ## Step 1: Lead Intake & Normalization |
| Normalize Lead Data8 | Set | Normalize Instagram payload into `body.*` | Incomming Lead instagram1 | Switch2 | ## Step 1: Lead Intake & Normalization |
| Normalize Lead Data9 | Set | Normalize LinkedIn payload into `body.*` | Incomming Lead linkdin1 | Switch2 | ## Step 1: Lead Intake & Normalization |
| Normalize Lead Data5 | Set | Normalize Website payload into `body.*` | Incomming Lead Website1 | Switch2 | ## Step 1: Lead Intake & Normalization |
| Switch2 | Switch | Channel routing placeholder (all go forward) | Normalize Lead Data5/6/7/8/9 | Extract Day and Hours1 | ## Step 1: Lead Intake & Normalization |
| Extract Day and Hours1 | Code | Convert to IST + working-hours evaluation | Switch2 | Is Working Day and Working Hour?1 | ## Step 2: Working Hours & AI Response |
| Is Working Day and Working Hour?1 | IF | Check working day + hours (both branches continue) | Extract Day and Hours1 | Code in JavaScript3 | ## Step 2: Working Hours & AI Response |
| Code in JavaScript3 | Code | Build Groq chat-completions request body | Is Working Day and Working Hour?1 | Get Ai Response1 | ## Step 2: Working Hours & AI Response |
| Get Ai Response1 | HTTP Request | Call Groq Llama chat completions | Code in JavaScript3 | Code in JavaScript4 | ## Step 2: Working Hours & AI Response |
| Code in JavaScript4 | Code | Extract AI text + merge normalized context | Get Ai Response1 | Switch3 | ## Step 2: Working Hours & AI Response |
| Switch3 | Switch | Route reply to correct sending node | Code in JavaScript4 | Send message / Send Instagram Message1 / Send Facebook Messages1 / Send Linkdin Messages1 / Send a message1 | ## Step 3: Send Reply & Log Data |
| Send message | WhatsApp | Send WhatsApp reply | Switch3 | Code in JavaScript5 | ## Step 3: Send Reply & Log Data |
| Send Instagram Message1 | HTTP Request | Send Instagram reply (placeholder) | Switch3 | Code in JavaScript5 | ## Step 3: Send Reply & Log Data |
| Send Facebook Messages1 | HTTP Request | Send Facebook reply (placeholder) | Switch3 | Code in JavaScript5 | ## Step 3: Send Reply & Log Data |
| Send Linkdin Messages1 | HTTP Request | Send LinkedIn reply (placeholder) | Switch3 | Code in JavaScript5 | ## Step 3: Send Reply & Log Data |
| Send a message1 | Gmail | Send email reply for website leads | Switch3 | Code in JavaScript5 | ## Step 3: Send Reply & Log Data |
| Code in JavaScript5 | Code | Determine send status + shape logging row | Any send node | google-sheet-name | ## Step 3: Send Reply & Log Data |
| google-sheet-name | Google Sheets | Append/update lead log in Sheet (match Email) | Code in JavaScript5 | — | ## Step 3: Send Reply & Log Data |
| Sticky Note | Sticky Note | Documentation block | — | — |  |
| Sticky Note8 | Sticky Note | Documentation block | — | — |  |
| Sticky Note9 | Sticky Note | Documentation block | — | — |  |
| Sticky Note10 | Sticky Note | Documentation block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create 5 Webhook trigger nodes (POST)**
   1) Webhook: **Incomming Lead whatsapp1**  
      - Path: `incoming-leads-whatsapp`, Method: POST  
   2) Webhook: **Incomming Lead facebook1**  
      - Path: `incoming-leads-facebook`, Method: POST  
   3) Webhook: **Incomming Lead instagram1**  
      - Path: `incoming-leads-instagram`, Method: POST  
   4) Webhook: **Incomming Lead linkdin1**  
      - Path: `incoming-leads-linkdin`, Method: POST  
   5) Webhook: **Incomming Lead Website1**  
      - Path: `incoming-leads-website`, Method: POST  
   - Activate or use “Test” URL while integrating each platform.

2. **Add 5 Set nodes to normalize payloads into a shared schema `body.*`**
   - **Normalize Lead Data6** (connect from WhatsApp webhook):  
     - `body.lead_name = {{$json.body.ProfileName}}`  
     - `body.contact_phone = {{$json.body.From}}`  
     - `body.message_text = {{$json.body.Body}}`  
     - `body.source_channel = whatsapp`  
     - `body.submitted_at = {{ new Date().toISOString() }}`  
   - **Normalize Lead Data7** (Facebook): same but source `facebook`
   - **Normalize Lead Data8** (Instagram): source `instagram`
   - **Normalize Lead Data9** (LinkedIn): source `linkdin` (or fix spelling everywhere if you change it)
   - **Normalize Lead Data5** (Website):  
     - map from `{{$json.body.lead_name}}`, `contact_phone`, `contact_email`, `message_text`, `source_channel`  
     - `body.submitted_at = {{ new Date().toISOString() }}`

3. **Create Switch node “Switch2” (channel classification)**
   - Value to evaluate: `{{$json.body.source_channel}}`
   - Add rules for whatsapp/Instagram/Facebook/LinkedIn/website (as in workflow)
   - Connect outputs of **all 5 Set nodes** into Switch2
   - Connect **each Switch2 output** to the next node (Extract Day and Hours1)
   - Important: decide on consistent casing/spelling (recommended) and make Switch values match your Set nodes.

4. **Add Code node “Extract Day and Hours1”**
   - Paste the provided IST conversion + working-hours JS logic.
   - Connect Switch2 → Extract Day and Hours1.

5. **Add IF node “Is Working Day and Working Hour?1”**
   - Conditions:
     - `{{$json.isWorkingDay}}` is true
     - `{{$json.is_working_hours}}` is true
   - (The original includes `ist_dayOfWeek < 6` too; optional)
   - Connect Extract Day and Hours1 → IF node
   - Connect both IF outputs (true/false) to **Code in JavaScript3** (matching the workflow behavior).

6. **Add Code node “Code in JavaScript3” (build Groq request)**
   - Create JS that builds:
     - `model: "llama-3.3-70b-versatile"`
     - `messages: [system, user]` with rules:
       - website → email style; else short chat
       - if outside working hours, mention office hours (9:30–6:30 IST)
       - do not mention AI
     - `temperature: 0.6`, `max_tokens: 160`
   - Output should merge original JSON and add `requestBody`.

7. **Add HTTP Request node “Get Ai Response1” (Groq)**
   - Method: POST
   - URL: `https://api.groq.com/openai/v1/chat/completions`
   - Send headers: true; Send body: true; Body type: JSON
   - Body: `{{$json.requestBody}}`
   - Headers:
     - `Authorization: Bearer <YOUR_GROQ_API_KEY>` (recommended format)
     - `Content-Type: application/json`
   - Connect Code in JavaScript3 → Get Ai Response1.

8. **Add Code node “Code in JavaScript4” (extract response)**
   - Extract: `choices[0].message.content`
   - Merge back to output object including:
     - `body`, `ai_response`, `is_working_hours`, IST fields
   - Connect Get Ai Response1 → Code in JavaScript4.

9. **Add Switch node “Switch3” (send routing)**
   - Evaluate `{{$json.body.source_channel}}`
   - Rules: whatsapp / Instagram / Facebook / LinkedIn / website
   - Connect Code in JavaScript4 → Switch3.

10. **Create sending nodes**
   - **WhatsApp node “Send message”**
     - Operation: send
     - Recipient: `{{$json.body.contact_phone}}`
     - Text: `{{$json.ai_response}}`
     - Configure WhatsApp credentials + phone number id
   - **HTTP Request “Send Instagram Message1”**
     - URL: your Instagram messaging endpoint
     - Add auth + body as required by your provider (not defined in workflow)
   - **HTTP Request “Send Facebook Messages1”**
     - URL: your Facebook messaging endpoint
     - Add auth + body as required
   - **HTTP Request “Send Linkdin Messages1”**
     - URL: your LinkedIn messaging endpoint
     - Add auth + body as required
   - **Gmail node “Send a message1”**
     - To: `{{$json.body.contact_email}}`
     - Subject: “Website developement Pricing Inquiry”
     - Message: `{{$json.ai_response}}`
     - Configure Gmail OAuth2 credentials

   Connect Switch3 outputs to the corresponding send node.

11. **Add Code node “Code in JavaScript5” (status + logging payload)**
   - Read send result from current `$input.item.json`
   - Pull original context from `$('Code in JavaScript4').item.json`
   - Build final log fields (timestamp, lead info, ai_response, working hour flags, status, response_sent_at)
   - Connect each send node → Code in JavaScript5.

12. **Add Google Sheets node “google-sheet-name”**
   - Operation: Append or Update
   - Document: select your Spreadsheet
   - Sheet: select the target sheet tab
   - Matching column: `Email`
   - Map columns:
     - Lead Name, Email, Phone, Lead Message, Timestamp, Source Channel, Working Hours, Day of Week, Response Time, AI Response, Status
   - Connect Code in JavaScript5 → Google Sheets.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Instant Reply SDR Worker (Multi-Channel Lead Auto-Responder)” description: triggers on channel webhooks, normalizes lead data, checks IST working hours, generates short human-like replies, routes back to channel, logs in Google Sheets. | Sticky note content embedded in workflow |
| Step 1: Lead Intake & Normalization | Sticky note content embedded in workflow |
| Step 2: Working Hours & AI Response | Sticky note content embedded in workflow |
| Step 3: Send Reply & Log Data | Sticky note content embedded in workflow |

**Key integration warning (important):** The workflow currently has inconsistent `source_channel` values (case differences like `instagram` vs `Instagram`, and spelling like `linkdin` vs `LinkedIn`). To ensure routing works, standardize the value set in normalization and the values checked in both Switch nodes.