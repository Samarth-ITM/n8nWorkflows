Send subscription renewal reminders via Telegram with Supabase

https://n8nworkflows.xyz/workflows/send-subscription-renewal-reminders-via-telegram-with-supabase-13193


# Send subscription renewal reminders via Telegram with Supabase

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow exposes a secured webhook endpoint that, when called, checks subscriptions stored in Supabase, identifies those expiring within **≤ 7 days**, sends **Telegram renewal reminders**, logs successful reminders in Supabase, logs failures/unauthorized requests to an error table, and returns a JSON response to the caller.

**Target use cases:**
- Daily renewal reminders triggered by a billing system, cron job, or monitoring service
- Lightweight “pull and notify” logic without needing a separate backend service
- Operational visibility through Supabase logging (success + errors)

### Logical blocks
1. **1.1 Input Reception & Authentication**
   - Receives POST webhook request, extracts API key, validates request.
2. **1.2 Subscription Fetching & Expiry Computation**
   - Lists subscriptions from Supabase, computes `daysToExpiry`.
3. **1.3 Filtering (Expiring within 7 days)**
   - Keeps only subscriptions expiring soon; otherwise ends with a success response.
4. **1.4 Telegram Notification**
   - Builds message, sends Telegram reminder, checks delivery result.
5. **1.5 Logging & Webhook Response (Success/Error)**
   - Logs successful reminders to `renewal_reminders`.
   - Logs errors (including unauthorized and Telegram failures) to `workflow_errors`.
   - Responds to the webhook caller.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Authentication
**Overview:** Exposes a POST webhook endpoint and enforces a shared-secret API key check before any database or Telegram actions run.

**Nodes involved:**
- Webhook Trigger
- Extract API Key
- Authorized?
- Compose Error Response (used by auth failure path)
- Insert Error Log (used by auth failure path)
- Respond to Webhook (used by auth failure path)

#### Webhook Trigger
- **Type / role:** `Webhook` — entry point for external calls.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `subscription-renewal`
  - **Response mode:** *Response Node* (workflow responds only via “Respond to Webhook” node)
- **Inputs / outputs:**
  - Output → **Extract API Key**
- **Version notes:** typeVersion `1.1`.
- **Edge cases / failures:**
  - Caller doesn’t send expected headers/body → downstream expressions may fail or treat as undefined.
  - If no Respond node runs on a branch, request could hang; here, error/success branches both connect to Respond.

#### Extract API Key
- **Type / role:** `Set` — intended to map request header `x-api-key` into a JSON field (e.g., `$json.apiKey`).
- **Key configuration choices (interpreted):**
  - Node currently shows **no explicit field mappings** in the JSON provided. For the IF check to work, this node should set something like:
    - `apiKey = {{$json.headers['x-api-key']}}` (exact path depends on n8n webhook payload structure/version).
- **Inputs / outputs:**
  - Input ← Webhook Trigger
  - Output → Authorized?
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If it does not actually set `apiKey`, then `$json.apiKey` will be undefined and **all requests will fail authorization**.
  - Header casing differences (`X-Api-Key` vs `x-api-key`) can break extraction unless handled.

#### Authorized?
- **Type / role:** `IF` — gatekeeper for API key validation.
- **Key configuration:**
  - Condition (String): `{{$json.apiKey}}` **equals** `{{YOUR_SECRET_KEY}}`
  - True branch continues; False branch goes to error handling.
- **Inputs / outputs:**
  - Input ← Extract API Key
  - **True** → Fetch Subscriptions
  - **False** → Compose Error Response
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Secret hardcoded in node (as shown) is a security risk if workflow is shared/exported. Prefer environment variables or n8n credentials/variables.
  - Timing attacks are not relevant here, but logging unauthorized requests is good practice.

---

### 2.2 Subscription Fetching & Expiry Computation
**Overview:** Pulls all subscription records from Supabase and enriches each row with a computed `daysToExpiry` integer.

**Nodes involved:**
- Fetch Subscriptions
- Calculate Days

#### Fetch Subscriptions
- **Type / role:** `Supabase` — reads subscription rows.
- **Key configuration:**
  - Operation: **List**
  - Table is not explicitly set in the node parameters in the JSON. Based on sticky note text, it is expected to list from **`subscriptions`**.
  - Requires Supabase credentials (Project URL + Service Role key recommended for server-side automation).
- **Inputs / outputs:**
  - Input ← Authorized? (true)
  - Output → Calculate Days
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - Missing/incorrect table selection → runtime error.
  - RLS (Row Level Security) can block reads unless service role key is used or policies allow.
  - Large tables: list operation may paginate/limit depending on node defaults; may miss records if not configured.

#### Calculate Days
- **Type / role:** `Code` — computes `daysToExpiry` per subscription.
- **Key configuration (logic):**
  - Uses current time: `const now = new Date();`
  - For each item:
    - `expiry = new Date(item.json.expiry_date)`
    - `diff = Math.ceil((expiry - now) / (1000*60*60*24))`
    - Adds `daysToExpiry: diff`
- **Inputs / outputs:**
  - Input ← Fetch Subscriptions (multiple items)
  - Output → Expiring Soon?
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Invalid/NULL `expiry_date` → `new Date(...)` becomes invalid; diff becomes `NaN`, breaking numeric comparisons later.
  - Timezone effects: dates without timezone info can shift day calculation; `Math.ceil` can produce “1 day left” earlier than expected.

---

### 2.3 Filtering (Expiring within 7 days)
**Overview:** Keeps only subscriptions with `daysToExpiry <= 7` for reminders. Non-expiring items go to a success response (effectively “nothing to do”).

**Nodes involved:**
- Expiring Soon?
- Compose Success Response (used by “not expiring” branch too)

#### Expiring Soon?
- **Type / role:** `IF` — filters items by expiry window.
- **Key configuration:**
  - Number condition: `{{$json.daysToExpiry}}` **smallerEqual** `7`
- **Inputs / outputs:**
  - Input ← Calculate Days
  - **True** → Create Telegram Message
  - **False** → Compose Success Response
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If `daysToExpiry` is `NaN`, comparison will evaluate false → item falls into “success/no action” even though data is broken.
  - Negative values (already expired) are also `<= 7` and will trigger reminders unless separately handled.

---

### 2.4 Telegram Notification
**Overview:** Creates a personalized Telegram message for each qualifying subscription and sends it through a Telegram bot.

**Nodes involved:**
- Create Telegram Message
- Send Telegram Reminder
- Telegram Sent?

#### Create Telegram Message
- **Type / role:** `Set` — constructs `telegramMessage` and ensures `chatId` is present for Telegram send.
- **Key configuration:**
  - Node parameters show no explicit mappings in the JSON provided, but it is expected to set fields such as:
    - `chatId = {{$json.telegram_chat_id}}`
    - `telegramMessage = ...` (using `customer_name`, `expiry_date`, `daysToExpiry`)
- **Inputs / outputs:**
  - Input ← Expiring Soon? (true)
  - Output → Send Telegram Reminder
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If `chatId` isn’t set correctly, Telegram node will fail (or send to wrong chat).
  - Message formatting issues (Markdown/HTML) aren’t configured; plain text safest.

#### Send Telegram Reminder
- **Type / role:** `Telegram` — sends a message.
- **Key configuration:**
  - `chatId`: `{{$json.chatId}}`
  - `text`: `{{$json.telegramMessage}}`
  - Uses Telegram API credential (bot token).
- **Inputs / outputs:**
  - Input ← Create Telegram Message
  - Output → Telegram Sent?
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - 403/401 if bot token invalid.
  - 400 if chat ID invalid or user hasn’t started the bot.
  - Rate limiting if many reminders are sent at once.

#### Telegram Sent?
- **Type / role:** `IF` — checks Telegram API response success.
- **Key configuration:**
  - Boolean condition: `{{$json.ok === true}}` isTrue
- **Inputs / outputs:**
  - Input ← Send Telegram Reminder
  - **True** → Prepare Reminder Log
  - **False** → Compose Error Response
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Telegram node output structure may differ; if `ok` is not present at `$json.ok`, condition may fail and route to error even on success.
  - Partial failures per item are handled per-item, which is appropriate.

---

### 2.5 Logging & Webhook Response (Success/Error)
**Overview:** Logs successful reminders to Supabase and logs errors (unauthorized, Telegram failures, other issues) to a `workflow_errors` table. All paths end by responding to the webhook.

**Nodes involved:**
- Prepare Reminder Log
- Insert Reminder Log
- Compose Success Response
- Compose Error Response
- Insert Error Log
- Respond to Webhook

#### Prepare Reminder Log
- **Type / role:** `Set` — prepares fields for insertion into `renewal_reminders`.
- **Key configuration:**
  - JSON shows no explicit mappings, but Insert Reminder Log expects:
    - `customer_id`
    - `reminder_sent_at` (ISO timestamp)
- **Inputs / outputs:**
  - Input ← Telegram Sent? (true)
  - Output → Insert Reminder Log
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If `reminder_sent_at` isn’t set, insert may fail if column is non-null without default.
  - If `customer_id` missing, insert may fail or log unusable record.

#### Insert Reminder Log
- **Type / role:** `Supabase` — inserts log row.
- **Key configuration:**
  - Table: `renewal_reminders`
  - Fields:
    - `customer_id = {{$json.customer_id}}`
    - `reminder_sent_at = {{$json.reminder_sent_at}}`
- **Inputs / outputs:**
  - Input ← Prepare Reminder Log
  - Output → Compose Success Response
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - RLS can block inserts unless service role or policies allow.
  - Duplicate logging if workflow is called multiple times per day and no uniqueness constraint exists.

#### Compose Success Response
- **Type / role:** `Set` — crafts a JSON response body for the webhook caller.
- **Key configuration:**
  - JSON shows no explicit response fields; expected to set something like `{ status: 'ok', processed: ..., reminded: ... }`.
- **Inputs / outputs:**
  - Input ← Insert Reminder Log (success path)
  - Also input ← Expiring Soon? (false path, i.e., nothing expiring)
  - Output → Respond to Webhook
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If used for both “no expiring items” and “reminder sent”, the response should distinguish cases; currently not guaranteed.

#### Compose Error Response
- **Type / role:** `Set` — creates a structured error payload and/or fields used by error logging.
- **Key configuration:**
  - No explicit mappings shown. Insert Error Log expects:
    - `message`
    - `error_detail`
- **Inputs / outputs:**
  - Input ← Authorized? (false) OR Telegram Sent? (false)
  - Output → Insert Error Log **and** Respond to Webhook (both connected)
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If it doesn’t set `message` / `error_detail`, error logs will be empty or inserts may fail if non-null constraints exist.
  - Because it connects to Respond to Webhook directly and also to Insert Error Log, timing can matter: the webhook may respond even if logging fails (depending on execution order and node behavior).

#### Insert Error Log
- **Type / role:** `Supabase` — inserts an error row.
- **Key configuration:**
  - Table: `workflow_errors`
  - Fields:
    - `error_message = {{$json.message}}`
    - `error_detail = {{$json.error_detail}}`
    - `created_at = {{ new Date().toISOString() }}`
- **Inputs / outputs:**
  - Input ← Compose Error Response
  - Output: none (terminal)
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - If error table doesn’t exist, workflow will fail while trying to log an error (secondary failure).
  - RLS may block inserts.

#### Respond to Webhook
- **Type / role:** `Respond to Webhook` — returns final HTTP response.
- **Key configuration:**
  - Uses default options (status code/body not explicitly set in JSON snippet).
  - Receives the current item JSON as response unless configured otherwise.
- **Inputs / outputs:**
  - Input ← Compose Success Response OR Compose Error Response
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - If it responds with an error structure but still returns HTTP 200 (default), callers may not detect failures unless they parse body.
  - Multiple items: Respond node generally returns a single response; ensure upstream paths consolidate or only provide one item if strict API contract is needed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | Webhook | Public entry point (POST) with response via Respond node | — | Extract API Key | ## Trigger & Authentication\nThis block handles all incoming requests. The **Webhook Trigger** node exposes a public endpoint that external services can call. Immediately after triggering, the workflow extracts the `x-api-key` header so it can be validated. The **Authorized?** IF node compares the provided key with your secret. Unauthorized traffic is short-circuited, returning an error JSON and recording the event in the error-logging branch. Keeping authentication logic up front prevents unnecessary Supabase queries and secures the rest of the workflow.\n\nKey points:\n• Webhook response mode is set to *Response Node* so we can craft custom replies.\n• API key travels in headers, avoiding exposure in query strings.\n• All failed checks are directed to the error-handling lane for visibility. |
| Extract API Key | Set | Extracts `x-api-key` into `$json.apiKey` (intended) | Webhook Trigger | Authorized? | ## Trigger & Authentication\nThis block handles all incoming requests. The **Webhook Trigger** node exposes a public endpoint that external services can call. Immediately after triggering, the workflow extracts the `x-api-key` header so it can be validated. The **Authorized?** IF node compares the provided key with your secret. Unauthorized traffic is short-circuited, returning an error JSON and recording the event in the error-logging branch. Keeping authentication logic up front prevents unnecessary Supabase queries and secures the rest of the workflow.\n\nKey points:\n• Webhook response mode is set to *Response Node* so we can craft custom replies.\n• API key travels in headers, avoiding exposure in query strings.\n• All failed checks are directed to the error-handling lane for visibility. |
| Authorized? | IF | Validates API key | Extract API Key | Fetch Subscriptions; Compose Error Response | ## Trigger & Authentication\nThis block handles all incoming requests. The **Webhook Trigger** node exposes a public endpoint that external services can call. Immediately after triggering, the workflow extracts the `x-api-key` header so it can be validated. The **Authorized?** IF node compares the provided key with your secret. Unauthorized traffic is short-circuited, returning an error JSON and recording the event in the error-logging branch. Keeping authentication logic up front prevents unnecessary Supabase queries and secures the rest of the workflow.\n\nKey points:\n• Webhook response mode is set to *Response Node* so we can craft custom replies.\n• API key travels in headers, avoiding exposure in query strings.\n• All failed checks are directed to the error-handling lane for visibility. |
| Fetch Subscriptions | Supabase | Lists subscription records | Authorized? | Calculate Days | ## Processing & Filtering\nOnce a request is authorized, the workflow pulls current subscription rows from Supabase. The **Calculate Days** code snippet computes the remaining time until each subscription’s `expiry_date`. Next, the **Expiring Soon?** IF node isolates customers whose plans lapse within seven days. This logic keeps Telegram traffic efficient, ensuring only relevant users receive messages. Because n8n processes items individually, each record moves independently through the remainder of the flow, enabling fine-grained logging and error handling without complex loops. |
| Calculate Days | Code | Computes `daysToExpiry` per subscription | Fetch Subscriptions | Expiring Soon? | ## Processing & Filtering\nOnce a request is authorized, the workflow pulls current subscription rows from Supabase. The **Calculate Days** code snippet computes the remaining time until each subscription’s `expiry_date`. Next, the **Expiring Soon?** IF node isolates customers whose plans lapse within seven days. This logic keeps Telegram traffic efficient, ensuring only relevant users receive messages. Because n8n processes items individually, each record moves independently through the remainder of the flow, enabling fine-grained logging and error handling without complex loops. |
| Expiring Soon? | IF | Filters subscriptions with `daysToExpiry <= 7` | Calculate Days | Create Telegram Message; Compose Success Response | ## Processing & Filtering\nOnce a request is authorized, the workflow pulls current subscription rows from Supabase. The **Calculate Days** code snippet computes the remaining time until each subscription’s `expiry_date`. Next, the **Expiring Soon?** IF node isolates customers whose plans lapse within seven days. This logic keeps Telegram traffic efficient, ensuring only relevant users receive messages. Because n8n processes items individually, each record moves independently through the remainder of the flow, enabling fine-grained logging and error handling without complex loops. |
| Create Telegram Message | Set | Prepares `chatId` and `telegramMessage` | Expiring Soon? | Send Telegram Reminder | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Send Telegram Reminder | Telegram | Sends Telegram reminder via bot | Create Telegram Message | Telegram Sent? | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Telegram Sent? | IF | Checks Telegram response success | Send Telegram Reminder | Prepare Reminder Log; Compose Error Response | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Prepare Reminder Log | Set | Prepares log payload for `renewal_reminders` | Telegram Sent? | Insert Reminder Log | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Insert Reminder Log | Supabase | Inserts reminder log row | Prepare Reminder Log | Compose Success Response | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Compose Success Response | Set | Creates success JSON for webhook caller | Expiring Soon?; Insert Reminder Log | Respond to Webhook | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Respond to Webhook | Respond to Webhook | Returns HTTP response to caller | Compose Success Response; Compose Error Response | — | ## How it works\nThis workflow listens for an incoming webhook call to kick off a subscription-renewal check. After validating an API key, it queries a Supabase table that stores all active subscriptions. Each record is enriched with a calculated **daysToExpiry** value, then filtered so only customers whose plan expires in seven days or fewer proceed. For every imminent renewal, a friendly Telegram reminder is sent. Successful sends are logged back to Supabase while failures or unauthorized calls create an error entry. Finally, the workflow returns a concise JSON response to the original webhook request.\n\n## Setup steps\n1. Add your Supabase project URL and service role key to an **Supabase API** credential.\n2. Create two tables: `subscriptions` (with columns: customer_id, customer_name, telegram_chat_id, expiry_date) and `renewal_reminders`.\n3. Generate a Telegram bot and store its token in a **Telegram API** credential.\n4. Replace `{{YOUR_SECRET_KEY}}` in the *Authorized?* IF node with a secure value and pass it via an `x-api-key` header when calling the webhook.\n5. Deploy the workflow and copy the production webhook URL from the *Webhook Trigger* node.\n6. Schedule or call the webhook from your billing system on a daily basis.\n7. Monitor Supabase tables for reminder logs and review Telegram delivery results. |
| Compose Error Response | Set | Creates error JSON and fields for error logging | Authorized?; Telegram Sent? | Insert Error Log; Respond to Webhook | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| Insert Error Log | Supabase | Inserts error log row into `workflow_errors` | Compose Error Response | — | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |
| 🔔 Subscription Renewal Reminder – Overview | Sticky Note | Documentation / overview | — | — | ## How it works\nThis workflow listens for an incoming webhook call to kick off a subscription-renewal check. After validating an API key, it queries a Supabase table that stores all active subscriptions. Each record is enriched with a calculated **daysToExpiry** value, then filtered so only customers whose plan expires in seven days or fewer proceed. For every imminent renewal, a friendly Telegram reminder is sent. Successful sends are logged back to Supabase while failures or unauthorized calls create an error entry. Finally, the workflow returns a concise JSON response to the original webhook request.\n\n## Setup steps\n1. Add your Supabase project URL and service role key to an **Supabase API** credential.\n2. Create two tables: `subscriptions` (with columns: customer_id, customer_name, telegram_chat_id, expiry_date) and `renewal_reminders`.\n3. Generate a Telegram bot and store its token in a **Telegram API** credential.\n4. Replace `{{YOUR_SECRET_KEY}}` in the *Authorized?* IF node with a secure value and pass it via an `x-api-key` header when calling the webhook.\n5. Deploy the workflow and copy the production webhook URL from the *Webhook Trigger* node.\n6. Schedule or call the webhook from your billing system on a daily basis.\n7. Monitor Supabase tables for reminder logs and review Telegram delivery results. |
| 🛡️ Trigger & Auth (Info) | Sticky Note | Documentation / block comment | — | — | ## Trigger & Authentication\nThis block handles all incoming requests. The **Webhook Trigger** node exposes a public endpoint that external services can call. Immediately after triggering, the workflow extracts the `x-api-key` header so it can be validated. The **Authorized?** IF node compares the provided key with your secret. Unauthorized traffic is short-circuited, returning an error JSON and recording the event in the error-logging branch. Keeping authentication logic up front prevents unnecessary Supabase queries and secures the rest of the workflow.\n\nKey points:\n• Webhook response mode is set to *Response Node* so we can craft custom replies.\n• API key travels in headers, avoiding exposure in query strings.\n• All failed checks are directed to the error-handling lane for visibility. |
| 📊 Processing (Info) | Sticky Note | Documentation / block comment | — | — | ## Processing & Filtering\nOnce a request is authorized, the workflow pulls current subscription rows from Supabase. The **Calculate Days** code snippet computes the remaining time until each subscription’s `expiry_date`. Next, the **Expiring Soon?** IF node isolates customers whose plans lapse within seven days. This logic keeps Telegram traffic efficient, ensuring only relevant users receive messages. Because n8n processes items individually, each record moves independently through the remainder of the flow, enabling fine-grained logging and error handling without complex loops. |
| 📨 Notification & Logging (Info) | Sticky Note | Documentation / block comment | — | — | ## Notifications & Logging\nFor every imminent renewal, a personalized message is assembled and dispatched via the **Send Telegram Reminder** node. Successful sends trigger a logging sequence that writes a confirmation row into `renewal_reminders`. If Telegram responds with an error (e.g., invalid chat ID), the alternative branch builds an error object and stores it in `workflow_errors`. Both success and failure paths conclude by assembling a human-readable JSON payload that the **Respond to Webhook** node returns to the caller, closing the loop with clear status feedback. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Subscription Renewal Reminder – Telegram & Supabase`
   - Execution order: default (`v1`)

2. **Add Webhook Trigger**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `subscription-renewal`
   - Response: **Using “Respond to Webhook” node** (Response Node mode)

3. **Add “Extract API Key”**
   - Node: **Set**
   - Add field:
     - `apiKey` = expression that reads request header `x-api-key`
       - Common options to try depending on your n8n version/webhook structure:
         - `{{$json.headers['x-api-key']}}`
         - `{{$json.headers['X-API-KEY']}}`
         - `{{$json.headers['x-api-key'] || $json.headers['X-Api-Key']}}`
   - Connect: Webhook Trigger → Extract API Key

4. **Add “Authorized?”**
   - Node: **IF**
   - Condition → **String**
     - Value 1: `{{$json.apiKey}}`
     - Operation: **equals**
     - Value 2: your secret (replace `{{YOUR_SECRET_KEY}}`)
   - Connect: Extract API Key → Authorized?

5. **Create Supabase credentials**
   - Credentials: **Supabase API**
   - Set:
     - **Project URL**
     - **Service Role key** (recommended for backend automation and bypassing RLS; secure it)
   - Ensure your Supabase tables exist:
     - `subscriptions` with columns: `customer_id`, `customer_name`, `telegram_chat_id`, `expiry_date`
     - `renewal_reminders`
     - `workflow_errors`

6. **Add “Fetch Subscriptions”**
   - Node: **Supabase**
   - Credential: your Supabase credential
   - Operation: **List**
   - Table: `subscriptions`
   - Connect: Authorized? (true) → Fetch Subscriptions

7. **Add “Calculate Days”**
   - Node: **Code**
   - Paste the provided JS logic to compute `daysToExpiry` from `expiry_date`
   - Connect: Fetch Subscriptions → Calculate Days

8. **Add “Expiring Soon?”**
   - Node: **IF**
   - Condition → **Number**
     - Value 1: `{{$json.daysToExpiry}}`
     - Operation: **smallerEqual**
     - Value 2: `7`
   - Connect: Calculate Days → Expiring Soon?

9. **Add “Create Telegram Message”**
   - Node: **Set**
   - Add fields (example structure):
     - `chatId` = `{{$json.telegram_chat_id}}`
     - `telegramMessage` = e.g. `Hi {{$json.customer_name}}, your subscription expires in {{$json.daysToExpiry}} day(s) ({{$json.expiry_date}}). Reply to renew.`
   - Connect: Expiring Soon? (true) → Create Telegram Message

10. **Create Telegram credentials**
   - Credentials: **Telegram API**
   - Add your bot token (BotFather)

11. **Add “Send Telegram Reminder”**
   - Node: **Telegram**
   - Resource/Operation: send message (default message send)
   - Chat ID: `{{$json.chatId}}`
   - Text: `{{$json.telegramMessage}}`
   - Connect: Create Telegram Message → Send Telegram Reminder

12. **Add “Telegram Sent?”**
   - Node: **IF**
   - Condition → **Boolean**
     - Value 1: `{{$json.ok === true}}`
     - Operation: **isTrue**
   - Connect: Send Telegram Reminder → Telegram Sent?

13. **Add “Prepare Reminder Log”**
   - Node: **Set**
   - Add fields:
     - `customer_id` = `{{$json.customer_id}}`
     - `reminder_sent_at` = `{{new Date().toISOString()}}`
   - Connect: Telegram Sent? (true) → Prepare Reminder Log

14. **Add “Insert Reminder Log”**
   - Node: **Supabase**
   - Credential: Supabase credential
   - Operation: **Insert**
   - Table: `renewal_reminders`
   - Map fields:
     - `customer_id = {{$json.customer_id}}`
     - `reminder_sent_at = {{$json.reminder_sent_at}}`
   - Connect: Prepare Reminder Log → Insert Reminder Log

15. **Add “Compose Success Response”**
   - Node: **Set**
   - Create a consistent response payload, for example:
     - `status = "ok"`
     - `message = "Processed request"`
     - optionally: include counts (requires additional aggregation logic if you want accurate counts)
   - Connect:
     - Insert Reminder Log → Compose Success Response
     - Expiring Soon? (false) → Compose Success Response

16. **Add “Compose Error Response”**
   - Node: **Set**
   - Add fields for both response and DB logging:
     - `message` = e.g. `"Unauthorized"` or `"Telegram send failed"`
     - `error_detail` = include context (chat id, telegram error description, etc.)
     - `status` = `"error"`
   - Connect:
     - Authorized? (false) → Compose Error Response
     - Telegram Sent? (false) → Compose Error Response

17. **Add “Insert Error Log”**
   - Node: **Supabase**
   - Operation: **Insert**
   - Table: `workflow_errors`
   - Map fields:
     - `error_message = {{$json.message}}`
     - `error_detail = {{$json.error_detail}}`
     - `created_at = {{new Date().toISOString()}}`
   - Connect: Compose Error Response → Insert Error Log

18. **Add “Respond to Webhook”**
   - Node: **Respond to Webhook**
   - Configure status codes if desired (recommended):
     - success: 200
     - unauthorized: 401
     - telegram failure: 502 or 400 depending on cause
   - Connect:
     - Compose Success Response → Respond to Webhook
     - Compose Error Response → Respond to Webhook

19. **Activate and test**
   - Call the production webhook URL with:
     - Header: `x-api-key: <your secret>`
   - Verify:
     - Telegram reminders delivered
     - `renewal_reminders` rows inserted
     - failures land in `workflow_errors`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create tables: `subscriptions` (customer_id, customer_name, telegram_chat_id, expiry_date) and `renewal_reminders` | From sticky note “🔔 Subscription Renewal Reminder – Overview” |
| Store Supabase project URL + service role key in a Supabase credential | From sticky note “🔔 Subscription Renewal Reminder – Overview” |
| Replace `{{YOUR_SECRET_KEY}}` and pass it via `x-api-key` header | From sticky note “🔔 Subscription Renewal Reminder – Overview” |
| Use webhook “Response Node” mode to craft custom replies and short-circuit unauthorized traffic | From sticky note “🛡️ Trigger & Auth (Info)” |
| Filtering logic: notify only if expiry is within 7 days | From sticky note “📊 Processing (Info)” |
| Log success to `renewal_reminders`; log failures to `workflow_errors` | From sticky note “📨 Notification & Logging (Info)” |