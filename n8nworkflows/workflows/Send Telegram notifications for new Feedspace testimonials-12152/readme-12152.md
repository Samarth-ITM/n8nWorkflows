Send Telegram notifications for new Feedspace testimonials

https://n8nworkflows.xyz/workflows/send-telegram-notifications-for-new-feedspace-testimonials-12152


# Send Telegram notifications for new Feedspace testimonials

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Feedspace Testimonial → Telegram Notification  
**Purpose:** Receive new testimonial events from Feedspace via an HTTP webhook, send a formatted Telegram message to a configured chat, then return an HTTP response indicating success (200) or failure (500).

**Target use cases**
- Businesses collecting testimonials in **Feedspace** who want **instant Telegram alerts**.
- Teams needing a lightweight “notify-on-new-testimonial” automation with basic error reporting to the webhook caller.

### Logical blocks
**1.1 Input Reception (Feedspace → n8n)**  
Receives a POST webhook from Feedspace and keeps the raw body for maximum compatibility.

**1.2 Notification Delivery (n8n → Telegram)**  
Builds a Markdown message using webhook payload fields (author name and feed URL when present) and sends it to a Telegram chat.

**1.3 Response Handling & Error Reporting**  
Checks whether the Telegram node produced an error object; returns either a success JSON response (200) or an error JSON response (500) with extracted details.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Feedspace → n8n)
**Overview:** Accepts incoming testimonial payloads from Feedspace through a dedicated POST webhook endpoint and forwards the parsed data to the notification step.

**Nodes involved**
- Receive Feedspace Webhook

#### Node: Receive Feedspace Webhook
- **Type / role:** `Webhook` — entry point that receives HTTP requests from Feedspace.
- **Key configuration (interpreted):**
  - **Method:** POST
  - **Path:** `feedspace-testimonial` (n8n will expose Test/Production URLs for this path)
  - **Raw body enabled:** `options.rawBody = true` (keeps raw request body available; helpful if signature validation or exact payload is needed later)
  - **Response mode:** `responseNode` (this node will *not* respond automatically; a Respond to Webhook node must respond)
- **Key expressions/variables:** None
- **Connections:**
  - **Output →** Send Telegram Notification
- **Version-specific notes:** Webhook node **typeVersion 2**; response handling uses Respond to Webhook nodes.
- **Potential failures / edge cases:**
  - Feedspace sends unexpected payload shape (missing `body.data...`)
  - Large payloads could hit n8n limits depending on instance configuration
  - If no Respond to Webhook executes (e.g., a hard crash), the caller may time out

---

### 2.2 Notification Delivery (n8n → Telegram)
**Overview:** Sends a Telegram message announcing a new testimonial, optionally including the author name and a link to the Feedspace feed.

**Nodes involved**
- Send Telegram Notification

#### Node: Send Telegram Notification
- **Type / role:** `Telegram` — sends a chat message via Telegram Bot API.
- **Key configuration (interpreted):**
  - **Operation:** Send Message (implied by `text` + `chatId`)
  - **Chat ID:** from workflow variable: `{{ $vars.TELEGRAM_CHAT_ID }}`
  - **Parse mode:** Markdown (`additionalFields.parse_mode = Markdown`)
  - **Attribution disabled:** `appendAttribution = false`
  - **Error handling:**
    - `onError: continueRegularOutput` and `continueOnFail: true`  
      This is critical: if Telegram fails (bad token, blocked bot, invalid chat ID), the workflow continues and exposes error data for downstream logic.
- **Message template (key expressions/variables):**
  - Builds a Markdown message:
    - Always includes: `🎉 *New Testimonial Received!*`
    - If present: `👤 *From:* <Name>` using `{{$json.body?.data?.response?.Name}}`
    - If present: `🔗 <feed_url>` using `{{$json.body?.data?.feed_url}}`
  - Uses optional chaining (`?.`) to avoid expression failures if fields are missing.
- **Connections:**
  - **Input ←** Receive Feedspace Webhook
  - **Output →** Has Error?
- **Credentials:**
  - Uses **Telegram account** credential (Bot token).
- **Version-specific notes:** Node **typeVersion 1.2**; Markdown parse mode must match Telegram formatting rules.
- **Potential failures / edge cases:**
  - **Auth/token invalid** → Telegram API returns error
  - **Chat ID invalid** or bot not added / blocked → send fails
  - Markdown formatting issues (rare, but can occur if user-provided text includes characters that break Markdown)
  - Payload doesn’t include expected `body.data...` fields; message still sends but may omit name/link

---

### 2.3 Response Handling & Error Reporting
**Overview:** Determines whether the Telegram step produced an error. If no error exists, returns a 200 success JSON to the webhook caller; otherwise formats error details and returns a 500 JSON.

**Nodes involved**
- Has Error?
- Success Response
- Format Error Details
- Error Response

#### Node: Has Error?
- **Type / role:** `IF` — branches based on whether an error exists in the current item.
- **Key configuration (interpreted):**
  - Condition checks existence of: `{{ $json.error }}`
  - Operator: “object exists”
- **Connections:**
  - **Input ←** Send Telegram Notification
  - **True/First output (no error path as wired):** Success Response  
    *(In this workflow, output index 0 goes to “Success Response”.)*
  - **False/Second output (error path as wired):** Format Error Details  
    *(Output index 1 goes to “Format Error Details”.)*
- **Version-specific notes:** IF node **typeVersion 2.2** with strict validation.
- **Potential failures / edge cases:**
  - If Telegram node fails in an unexpected shape and doesn’t set `$json.error`, the IF may misclassify outcome.
  - If multiple items were produced (not typical here), only per-item branching applies.

#### Node: Success Response
- **Type / role:** `Respond to Webhook` — sends HTTP 200 JSON back to Feedspace webhook request.
- **Key configuration (interpreted):**
  - Response code: **200**
  - Body: JSON object:
    - `status: "success"`
    - `message: "Notification sent successfully"`
    - `timestamp: $now.toISO()`
- **Connections:**
  - **Input ←** Has Error? (success branch)
  - **Output:** none (terminates request)
- **Potential failures / edge cases:**
  - If executed after a long delay, the original webhook caller may have already timed out.

#### Node: Format Error Details
- **Type / role:** `Code` — normalizes error details into a consistent JSON payload.
- **Key configuration (interpreted):**
  - Reads the first input item: `const inputData = $input.first().json;`
  - Extracts:
    - `errorMessage = inputData.error?.message || 'Unknown error occurred'`
    - `errorCode = inputData.error?.code || 'UNKNOWN'`
  - Returns:
    - `status: "error"`
    - `code`
    - `message`
    - `timestamp` (ISO string)
- **Connections:**
  - **Input ←** Has Error? (error branch)
  - **Output →** Error Response
- **Version-specific notes:** Code node **typeVersion 2**.
- **Potential failures / edge cases:**
  - If the error is not stored under `json.error`, output becomes `UNKNOWN` / “Unknown error occurred”
  - If there is no input item (unexpected), `$input.first()` may throw (rare in this linear flow)

#### Node: Error Response
- **Type / role:** `Respond to Webhook` — sends HTTP 500 JSON back to the webhook caller with formatted error details.
- **Key configuration (interpreted):**
  - Response code: **500**
  - Body: `{{ $json }}` (passes through the formatted error object)
- **Connections:**
  - **Input ←** Format Error Details
  - **Output:** none (terminates request)
- **Potential failures / edge cases:**
  - Same timeout consideration as success response

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Feedspace Webhook | Webhook | Receives Feedspace testimonial webhook (POST) | — | Send Telegram Notification | ## Webhook & Notification / Receives Feedspace webhook and sends Telegram notification |
| Send Telegram Notification | Telegram | Sends formatted Telegram message to configured chat | Receive Feedspace Webhook | Has Error? | ## Webhook & Notification / Receives Feedspace webhook and sends Telegram notification |
| Has Error? | IF | Checks whether Telegram produced an error object | Send Telegram Notification | Success Response; Format Error Details | ## Response Handling / Checks for errors and returns appropriate HTTP response |
| Success Response | Respond to Webhook | Returns HTTP 200 JSON success payload | Has Error? | — | ## Response Handling / Checks for errors and returns appropriate HTTP response |
| Format Error Details | Code | Extracts/normalizes error details for response | Has Error? | Error Response | ## Response Handling / Checks for errors and returns appropriate HTTP response |
| Error Response | Respond to Webhook | Returns HTTP 500 JSON error payload | Format Error Details | — | ## Response Handling / Checks for errors and returns appropriate HTTP response |
| Sticky Note | Sticky Note | Documentation / usage notes | — | — | ## 🔔 Feedspace → Telegram Notifications / @[youtube](AkjdEv5t50A) / Businesses using [Feedspace](https://feedspace.io) to collect testimonials who want instant Telegram notifications. / Setup: [@BotFather](https://t.me/BotFather), [@userinfobot](https://t.me/userinfobot) |
| Sticky Note1 | Sticky Note | Visual grouping label | — | — | ## Webhook & Notification / Receives Feedspace webhook and sends Telegram notification |
| Response Handling Group | Sticky Note | Visual grouping label | — | — | ## Response Handling / Checks for errors and returns appropriate HTTP response |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Feedspace Testimonial → Telegram Notification**
   - Ensure workflow settings use standard execution order (default is fine).

2. **Add node: Webhook**
   - Node type: **Webhook**
   - Name: **Receive Feedspace Webhook**
   - Set:
     - **HTTP Method:** POST
     - **Path:** `feedspace-testimonial`
     - **Options:** enable **Raw Body**
     - **Response Mode:** **Using “Respond to Webhook” node** (responseNode)
   - This node will be the workflow entry point.

3. **Add node: Telegram**
   - Node type: **Telegram**
   - Name: **Send Telegram Notification**
   - Credentials:
     - Create Telegram credentials using a bot token from **@BotFather**.
   - Parameters:
     - **Chat ID:** `{{ $vars.TELEGRAM_CHAT_ID }}`
     - **Text:** use a Markdown-formatted expression similar to:
       - Title line: `🎉 *New Testimonial Received!*`
       - Optional name: from `{{$json.body?.data?.response?.Name}}`
       - Optional link: from `{{$json.body?.data?.feed_url}}`
     - **Additional Fields → Parse Mode:** Markdown
     - **Additional Fields → Append Attribution:** false
   - Node error behavior:
     - Enable “Continue on Fail” (so the next IF node can detect errors)
     - Set node’s error handling to continue regular output (if available in your n8n UI/version).

4. **Connect nodes**
   - Connect **Receive Feedspace Webhook → Send Telegram Notification**

5. **Add node: IF**
   - Node type: **IF**
   - Name: **Has Error?**
   - Condition:
     - Check whether `{{ $json.error }}` **exists** (object exists)
   - Connect **Send Telegram Notification → Has Error?**

6. **Add node: Respond to Webhook (success)**
   - Node type: **Respond to Webhook**
   - Name: **Success Response**
   - Configure:
     - **Response Code:** 200
     - **Respond With:** JSON
     - **Response Body** (expression) returns:
       - `status = "success"`
       - `message = "Notification sent successfully"`
       - `timestamp = $now.toISO()`
   - Connect **Has Error? (branch without error as wired in your logic) → Success Response**  
   - Note: In your current workflow wiring, the first output goes to success. Ensure your IF branches match your intended semantics.

7. **Add node: Code (error formatting)**
   - Node type: **Code**
   - Name: **Format Error Details**
   - Code logic:
     - Read `$input.first().json`
     - Extract `error.message` and `error.code`
     - Output a normalized JSON containing `status`, `code`, `message`, `timestamp`
   - Connect **Has Error? (error branch) → Format Error Details**

8. **Add node: Respond to Webhook (error)**
   - Node type: **Respond to Webhook**
   - Name: **Error Response**
   - Configure:
     - **Response Code:** 500
     - **Respond With:** JSON
     - **Response Body:** `{{ $json }}` (pass-through from Format Error Details)
   - Connect **Format Error Details → Error Response**

9. **Create required workflow variable**
   - In n8n, create a variable:
     - Name: `TELEGRAM_CHAT_ID`
     - Value: your numeric chat ID (from **@userinfobot**)
   - Ensure the workflow has access to this variable (environment/instance/workflow variables depending on your n8n setup).

10. **Configure Feedspace**
   - Activate the workflow.
   - Copy the **Production Webhook URL** from the Webhook node.
   - In Feedspace: **Automations → Webhook**, paste the production URL and set method to POST (as required).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video reference: `@[youtube](AkjdEv5t50A)` | Included in sticky note (YouTube embed reference) |
| Feedspace product link | https://feedspace.io |
| Create Telegram bot via BotFather (`/newbot`) | https://t.me/BotFather |
| Retrieve numeric Chat ID via userinfobot | https://t.me/userinfobot |
| Setup reminder: add Telegram credentials and create variable `TELEGRAM_CHAT_ID` | Sticky note setup steps |
| Feedspace configuration location: Automations → Webhook; use Production webhook URL | Sticky note setup steps |