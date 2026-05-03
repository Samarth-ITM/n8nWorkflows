Book WhatsApp consultations and sync contacts with Airtable and Google Calendar

https://n8nworkflows.xyz/workflows/book-whatsapp-consultations-and-sync-contacts-with-airtable-and-google-calendar-12763


# Book WhatsApp consultations and sync contacts with Airtable and Google Calendar

## 1. Workflow Overview

**Purpose:**  
This workflow connects a WhatsApp Business webhook to an intent-classifying AI agent and a WhatsApp Flow-based booking journey. It returns Flow screens (service selection → date/time selection → confirmation) with live Google Calendar availability, then on completion **creates/updates records in Airtable** and **creates a Google Calendar event**, and finally sends a WhatsApp confirmation message.

**Primary use cases:**
- Reply to normal WhatsApp text messages with either:
  - a helpful free-text answer, or
  - a WhatsApp template that launches a booking Flow.
- Power a WhatsApp Flow “consultation booking” experience with encrypted request/response handling.
- Persist contacts + bookings in Airtable and keep them synced to Google Calendar.

### Logical Blocks
**1.1 Webhook verification (GET)**  
Meta/WhatsApp webhook verification handshake.

**1.2 Webhook reception + request parsing (POST)**  
Single POST endpoint receives both normal WhatsApp messages and encrypted WhatsApp Flow requests; it routes them accordingly.

**1.3 Regular message handling + AI intent classification**  
Text messages go to an n8n LangChain Agent using OpenAI; the agent chooses between tools to send either text replies or the “booking” template.

**1.4 WhatsApp Flow runtime (encrypted)**  
Encrypted Flow requests are decrypted, routed by `action/screen/action_type`, then re-encrypted and returned to WhatsApp.

**1.5 Calendar availability (Google Calendar API)**  
Build available slots from Google Calendar events for “today” and for a selected date refresh.

**1.6 Booking finalization (Airtable + Google Calendar + WhatsApp confirmation)**  
On Flow completion: compute booking times, upsert customer, create booking, create calendar event, update booking, send WhatsApp confirmation, then return Flow success screen.

---

## 2. Block-by-Block Analysis

### 2.1 Webhook verification (GET)

**Overview:**  
Handles Meta webhook verification (`hub.challenge`) by checking a verify token and returning the challenge as plaintext.

**Nodes involved:**
- **GET: Verify Webhook**
- **Verify Token**
- **Return Challenge**

#### Node: GET: Verify Webhook
- **Type / role:** Webhook trigger (GET) for Meta verification.
- **Key configuration:**
  - Path: `whatsapp-webhook`
  - Response mode: “responseNode” (the response is sent by a Respond to Webhook node).
- **Input/Output:**
  - Output → **Verify Token**
- **Failure modes / edge cases:**
  - If Meta calls with missing query params, downstream code may throw.

**Sticky Note (applies):**  
“## Utility: WhatsApp Webhook Check  
Required for setting up your n8n webhook in Meta”

#### Node: Verify Token
- **Type / role:** Code node validating `hub.mode` and `hub.verify_token`.
- **Key configuration choices:**
  - Reads verification token from environment: `WHATSAPP_VERIFY_TOKEN`, fallback `'your_verify_token_here'`.
  - Validates:
    - `hub.mode === 'subscribe'`
    - `hub.verify_token === VERIFY_TOKEN`
  - Returns `{ challenge: hub.challenge }`.
- **Input/Output:**
  - Input: Webhook query from **GET: Verify Webhook**
  - Output → **Return Challenge**
- **Failure modes / edge cases:**
  - Throws error if mode/token invalid (Meta verification fails).
  - If `$env.WHATSAPP_VERIFY_TOKEN` isn’t set and you didn’t replace the fallback, verification will fail.

#### Node: Return Challenge
- **Type / role:** Respond to Webhook (plaintext).
- **Key configuration:**
  - Response code: 200
  - Respond with: `text`
  - Body: `{{ $json.challenge }}`
- **Input/Output:**
  - Input from **Verify Token**
  - Ends request.

---

### 2.2 Webhook reception + request parsing (POST)

**Overview:**  
Receives incoming WhatsApp webhook POSTs. It determines whether the payload is a normal WhatsApp message/status update or an encrypted Flow request, and normalizes the output for routing.

**Nodes involved:**
- **POST: Receive Messages1**
- **Decrypt WhatsApp Request1**
- **Is Regular Message?**

#### Node: POST: Receive Messages1
- **Type / role:** Webhook trigger (POST) for all inbound WhatsApp webhook payloads.
- **Key configuration:**
  - Path: `whatsapp-webhook` (same as GET; WhatsApp constraint)
  - HTTP Method: POST
  - Response mode: responseNode
- **Input/Output:**
  - Output → **Decrypt WhatsApp Request1**
- **Failure modes / edge cases:**
  - If WhatsApp retries, you must ensure idempotency downstream (this workflow partially acknowledges early for regular messages).

**Sticky Note (applies):**  
“## Whatsapp Single Entry Point: Webhook  
Because of Meta's Single App to Single Webhook restriction, all messages come through the same webhook (GET / POST).  
Conditional statements after this are then used to filter what happens”

#### Node: Decrypt WhatsApp Request1
- **Type / role:** Code node that:
  - Parses regular WhatsApp message webhook structure, OR
  - Decrypts encrypted WhatsApp Flow request payload (RSA → AES-128-GCM).
- **Key configuration choices:**
  - Uses env:
    - `WHATSAPP_PRIVATE_KEY`
    - `WHATSAPP_PRIVATE_KEY_PASSPHRASE`
  - Fixes private key formatting if stored with escaped newlines (`\\n`).
  - Regular message detection: checks `body.entry` presence.
  - Regular message normalization:
    - **Text messages:** outputs `messageType: 'text_message'`, `from`, `text`, `isRegularMessage: true`.
    - **Interactive button replies:** outputs `messageType: 'button_reply'`, `action`, `recordId`, etc., `isRegularMessage: true`.
    - **Other types:** outputs `messageType: <type>`, raw message, `isRegularMessage: true`.
    - **Status updates / non-message entry:** outputs `messageType: 'status_update'`, raw body.
  - Flow decryption:
    - RSA OAEP SHA-256 decrypt `encrypted_aes_key`
    - AES-128-GCM decrypt `encrypted_flow_data` with `initial_vector`
    - Returns `decrypted` payload + `aesKeyBase64`, `ivBase64`
  - Special Flow action: `ping` is handled here by immediately creating an encrypted `response` field.
- **Input/Output:**
  - Input: `body` from webhook
  - Output → **Is Regular Message?**
- **Failure modes / edge cases:**
  - Missing/invalid private key or passphrase → `crypto.privateDecrypt` throws.
  - Invalid base64 input → Buffer conversion errors.
  - AES auth tag mismatch → `decipher.final()` throws.
  - WhatsApp message types not explicitly handled may be output with `raw` but routed unexpectedly.
  - Naming inconsistency risk: this node emits `messageType: 'status_update'` (underscore), but downstream Switch checks for `status_updated` (past tense), see Block 2.3.

#### Node: Is Regular Message?
- **Type / role:** IF node routing between regular WhatsApp messages and Flow actions.
- **Key configuration:**
  - Condition: `$json.isRegularMessage === true`
- **Input/Output:**
  - **True** → **Switch** (regular message routing/AI)
  - **False** → **Switch on Action** (Flow runtime)
- **Failure modes / edge cases:**
  - If `isRegularMessage` is missing, condition fails and routes to Flow path (likely wrong).

---

### 2.3 Regular message handling + AI intent classification

**Overview:**  
For normal WhatsApp messages, the workflow quickly responds 200 OK to WhatsApp, and (for text messages) uses an AI agent to decide whether to send a booking template (Flow launcher) or a normal text response.

**Nodes involved:**
- **Switch**
- **Return 200 OK (Regular Message)**
- **Whatsapp Agent**
- **OpenAI Chat Model**
- **whatsapp_message_tool**
- **whatsapp_consult_template**

#### Node: Switch
- **Type / role:** Switch on normalized `messageType`.
- **Key configuration (rules):**
  - INTERACTIVE if `messageType == 'interactive'`
  - STATUS_UPDATE if `messageType == 'status_updated'`
  - TEXT_MESSAGE if `messageType == 'text_message'`
- **Input/Output:**
  - INTERACTIVE → **Return 200 OK (Regular Message)**
  - STATUS_UPDATE → **Return 200 OK (Regular Message)**
  - TEXT_MESSAGE → **Whatsapp Agent**
- **Edge cases / likely issues:**
  - **Mismatch 1:** Decrypt node emits `messageType: 'status_update'`, but Switch checks `'status_updated'` → status updates may not match any rule (and may stall).
  - **Mismatch 2:** Decrypt node emits interactive button replies as `messageType: 'button_reply'`, but Switch checks `'interactive'` → button replies won’t match.
  - If no rule matches, execution ends without responding (WhatsApp will retry).

**Sticky Note (applies):**  
“## Intelligent Templating Assignment & Responses  
We want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.  
If they want to book - then a Calendar Flow, a quote, then a quote flow.”

#### Node: Return 200 OK (Regular Message)
- **Type / role:** Respond to Webhook (JSON) to acknowledge receipt quickly.
- **Key configuration:**
  - HTTP 200
  - JSON body: `{"status":"received"}`
- **Input/Output:**
  - Used as terminal response for regular-message paths.
- **Failure modes / edge cases:**
  - If the workflow routes incorrectly and never reaches a Respond node, WhatsApp webhook will time out/retry.

#### Node: Whatsapp Agent
- **Type / role:** LangChain Agent that decides which WhatsApp sending tool to call.
- **Key configuration choices:**
  - Input text: `{{$json.text}}`
  - System message defines role, tools, examples, and a strict rule: tool output must be plain text for `whatsapp_message`.
- **Tools connected:**
  - **whatsapp_consult_template** (send booking template with Flow button)
  - **whatsapp_message_tool** (send plain text)
- **Model:**
  - Uses **OpenAI Chat Model** node as its language model.
- **Input/Output:**
  - Input from **Switch** (TEXT_MESSAGE)
  - Output → **Return 200 OK (Regular Message)** (the actual sending happens inside tool calls; agent output then is acknowledged)
- **Failure modes / edge cases:**
  - If OpenAI credentials/model fail, message won’t be sent but webhook still may get 200 (depending on execution order).
  - The system prompt includes placeholder “[Your Company Name]”; leaving it unchanged can reduce response quality/branding.

#### Node: OpenAI Chat Model
- **Type / role:** LLM provider for the agent.
- **Key configuration:**
  - Model: `gpt-4o`
- **Connections:**
  - AI language model output → **Whatsapp Agent**
- **Failure modes:**
  - Missing OpenAI credentials, quota exhaustion, or model unavailability.

#### Node: whatsapp_message_tool
- **Type / role:** HTTP Request Tool (LangChain tool) to send a WhatsApp text message.
- **Key configuration:**
  - POST `https://graph.facebook.com/v23.0/{{WHATSAPP_PHONE_NUMBER_ID}}/messages`
  - Auth: Bearer (generic credential)
  - Body `to`: `{{ $('Is Regular Message?').item.json.from }}`
  - Text body: `{{ $fromAI('JSON', ``, 'json') }}`
- **Edge cases / failure modes:**
  - Requires a valid bearer token with WhatsApp messaging permissions.
  - `$fromAI(...)` expects the agent to output valid content; if the agent returns non-text/object, request may send malformed body.
  - Uses `Is Regular Message?` item context; if multiple items or different branching occurs, `item.json.from` may be missing.

#### Node: whatsapp_consult_template
- **Type / role:** HTTP Request Tool to send a WhatsApp template that includes a Flow button.
- **Key configuration:**
  - Template name: `personal_consultation_booking`
  - Button: subtype `flow`, sends parameter `flow_token` set to `{{ $json.from }}`
  - `to`: `{{ $('Is Regular Message?').item.json.from }}`
- **Edge cases / failure modes:**
  - Template must exist, be approved, and match language code `en`.
  - Flow must be attached to the template button in WhatsApp Manager.
  - Ensure `WHATSAPP_PHONE_NUMBER_ID` is available (it is referenced as `{{WHATSAPP_PHONE_NUMBER_ID}}`, which in n8n typically needs to be an expression like `$env.WHATSAPP_PHONE_NUMBER_ID` or a fixed value unless templating is handled externally).

---

### 2.4 WhatsApp Flow runtime routing (decrypted → switch → response building)

**Overview:**  
Routes decrypted Flow actions (`ping`, `INIT`, `SERVICE_SELECTION`, `DATE_REFRESH`, `CONFIRM_BOOKING`, `COMPLETE_BOOKING`) to the appropriate screen-building logic, then merges responses into a single encryption+respond step.

**Nodes involved:**
- **Switch on Action**
- **Return Ping Response**
- **Handle INIT - Return Consultation Types**
- **Prepare Calendar Request**
- **Google Calendar Events**
- **Calculate Available Slots**
- **Prepare Date Refresh Request1**
- **Google Calendar Events (Date)**
- **Calculate Slots for Date**
- **Handle Confirm Booking**
- **Prepare Booking Data1**
- **Merge Paths**
- **Encrypt Response**
- **Respond to Webhook**

#### Node: Switch on Action
- **Type / role:** Switch for Flow actions and screens.
- **Key routing rules:**
  - PING: `$json.action == 'ping'`
  - INIT: `$json.action == 'INIT'`
  - SERVICE_SELECTION: `$json.action == 'data_exchange' && $json.screen == 'SERVICE_SELECTION'`
  - DATE_REFRESH: `$json.action == 'data_exchange' && $json.screen == 'DATE_TIME_SELECTION' && $json.data.action_type == 'refresh_slots'`
  - CONFIRM_BOOKING: `$json.action == 'data_exchange' && $json.screen == 'DATE_TIME_SELECTION' && $json.data.action_type == 'confirm_booking'`
  - COMPLETE_BOOKING: `$json.action == 'data_exchange' && $json.screen == 'CONFIRMATION' && $json.data.action_type == 'complete_booking'`
- **Connections:**
  - PING → **Return Ping Response**
  - INIT → **Handle INIT - Return Consultation Types**
  - SERVICE_SELECTION → **Prepare Calendar Request**
  - DATE_REFRESH → **Prepare Date Refresh Request1**
  - CONFIRM_BOOKING → **Handle Confirm Booking**
  - COMPLETE_BOOKING → **Prepare Booking Data1**
- **Edge cases / failure modes:**
  - If a Flow sends unexpected `screen` or `action_type`, no route matches → no response (Flow breaks).
  - The switch uses “fallbackOutput: none”; unmatched requests drop.

**Sticky Note (applies):**  
“## WhatsApp Flow: Booking  
Flows are attached to messaging templates and are activated by the WhatsApp User  
At each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made”

#### Node: Return Ping Response
- **Type / role:** RespondToWebhook for ping (already encrypted by decrypt node).
- **Key configuration:** Respond with text: `{{$json.response}}`
- **Input/Output:** Terminal response.
- **Edge cases:**
  - This is a special path: encryption is done inside **Decrypt WhatsApp Request1** for ping. Other Flow paths encrypt later.

#### Node: Handle INIT - Return Consultation Types
- **Type / role:** Code node building the initial Flow screen payload.
- **Key configuration choices:**
  - Sets `screen: 'SERVICE_SELECTION'`
  - Provides `consultation_types` (30/60 min)
  - Sets min/max date (today to +30 days)
  - Passes through `aesKeyBase64` and `ivBase64` for later encryption
- **Output →** **Merge Paths**
- **Edge cases:**
  - Date formatting uses `toISOString().split('T')[0]` (UTC date). If your business timezone differs, date boundaries can be off by one day.

#### Node: Handle Confirm Booking
- **Type / role:** Code node building the “CONFIRMATION” screen content based on selected details.
- **Key configuration:**
  - `screen: 'CONFIRMATION'`
  - Copies fields from `$json.data`
  - Passes through encryption keys
- **Output →** **Merge Paths**
- **Edge cases:**
  - If required fields (name/date/time) are missing, confirmation screen will be incomplete.

#### Node: Merge Paths
- **Type / role:** Code node to unify multiple branches before encryption.
- **Key configuration:** `return $input.all();`
- **Input/Output:**
  - Receives items from multiple Flow branches
  - Output → **Encrypt Response**
- **Edge cases:**
  - If multiple branches accidentally fire (shouldn’t happen with Switch), this will pass multiple items; encryption node will run per-item, but RespondToWebhook can only respond once reliably.

#### Node: Encrypt Response
- **Type / role:** Code node encrypting Flow response (AES-128-GCM) using the provided `aesKeyBase64` and `ivBase64`.
- **Key configuration:**
  - Flips IV bytes (`~iv[i] & 0xff`) as required by WhatsApp Flow encryption scheme.
  - Expects input JSON to contain:
    - `response` (object)
    - `aesKeyBase64` (string)
    - `ivBase64` (string)
  - Output: `{ encryptedResponse }` (base64 text)
- **Output →** **Respond to Webhook**
- **Failure modes:**
  - Missing keys or invalid base64 → encryption errors.
  - Response object not serializable → JSON stringify error.

#### Node: Respond to Webhook
- **Type / role:** RespondToWebhook returning encrypted Flow payload.
- **Key configuration:**
  - Content-Type: `text/plain`
  - Body: `{{$json.encryptedResponse}}`
- **Terminal response** for most Flow actions.

---

### 2.5 Calendar availability (Google Calendar)

**Overview:**  
Generates available slots by fetching Google Calendar events and calculating gaps within business hours (09:00–17:00, weekdays only). There are two paths: initial “today” slots and a “selected date refresh” slots.

**Nodes involved:**
- **Prepare Calendar Request**
- **Google Calendar Events**
- **Calculate Available Slots**
- **Prepare Date Refresh Request1**
- **Google Calendar Events (Date)**
- **Calculate Slots for Date**

#### Node: Prepare Calendar Request
- **Type / role:** Code node preparing a Google Calendar Events API query for “now until end of day”.
- **Key configuration:**
  - TIMEZONE hardcoded: `Europe/Amsterdam`
  - `timeMin = now.toISOString()`, `timeMax = endOfDay.toISOString()`
  - Calendar ID from env `GOOGLE_CALENDAR_ID` (fallback `primary`)
  - Passes consultation type + customer data from Flow payload
- **Output →** **Google Calendar Events**
- **Edge cases:**
  - Uses ISO times (UTC); also passes `timeZone` parameter later, which helps, but be consistent across computations.
  - If `GOOGLE_CALENDAR_ID` env and node calendar selection differ, you may query one calendar but create events in another.

#### Node: Google Calendar Events
- **Type / role:** HTTP Request to Google Calendar v3 Events list endpoint.
- **Key configuration:**
  - URL: `https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events`
  - Auth: Google Calendar OAuth2 (predefined credential)
  - Query: `timeMin`, `timeMax`, `timeZone`, `singleEvents=true`, `orderBy=startTime`, `maxResults=250`
- **Output →** **Calculate Available Slots**
- **Failure modes:**
  - OAuth token expired/invalid, insufficient scopes, calendar not shared, rate limiting.

#### Node: Calculate Available Slots
- **Type / role:** Code node building up to 8 available slots for today.
- **Key configuration choices:**
  - Business hours: 09:00–17:00
  - Slot step: 30 minutes
  - Duration: 30 or 60 depending on `consultationType`
  - Excludes weekends
  - Filters out cancelled events; blocks time for other statuses too
  - Uses timezone-aware “current time” calculation via `Intl.DateTimeFormat(timeZone: TIMEZONE)`
  - Returns Flow screen `DATE_TIME_SELECTION` with `available_slots`
- **Output →** **Merge Paths**
- **Edge cases:**
  - All-day events treated as busy for their date range (blocks the whole day).
  - Event dateTime parsing uses `new Date(event.start.dateTime)`; ensure Google returns RFC3339 with timezone offsets (it does).
  - The min/max date computation formats with locale `en-CA` and replaces `/` with `-`; depending on runtime locale output, verify formatting stays `YYYY-MM-DD`.

#### Node: Prepare Date Refresh Request1
- **Type / role:** Code node preparing Events API query for a selected date (00:00:00–23:59:59.999).
- **Key configuration:**
  - `selected_date` taken from `$json.data.selected_date`
  - `timeMin/timeMax` computed for that day in local JS Date (no explicit timezone)
  - Calendar ID from env `GOOGLE_CALENDAR_ID` fallback `primary`
- **Output →** **Google Calendar Events (Date)**
- **Edge cases:**
  - Day boundaries are computed in server timezone (unless the server is set appropriately). If your server timezone differs from the calendar’s timezone, slot calculations can drift.

#### Node: Google Calendar Events (Date)
- **Type / role:** HTTP Request (same endpoint) for the selected date window.
- **Key configuration:**
  - Query: `timeMin`, `timeMax`, `singleEvents=true`, `orderBy=startTime`, `maxResults=50`
  - Note: no `timeZone` query param here (unlike the “today” request).
- **Output →** **Calculate Slots for Date**
- **Failure modes:** same as above.

#### Node: Calculate Slots for Date
- **Type / role:** Code node generating all available slots for the selected date.
- **Key configuration choices:**
  - Business hours: 09:00–17:00, weekdays only
  - Step: 30 minutes
  - Skips slots in the past (`slotStart <= now`)
  - Considers any non-cancelled event as conflict
  - Returns `DATE_TIME_SELECTION` screen with `available_slots`
- **Output →** **Merge Paths**
- **Edge cases:**
  - Uses `now = new Date()` (server timezone) to compare slot start times created from `selectedDateStr + 'T00:00:00'` (server timezone). This is not timezone-safe if your server timezone differs from the business timezone.

---

### 2.6 Booking finalization (Airtable + Google Calendar + WhatsApp confirmation + Flow success)

**Overview:**  
On Flow completion, builds booking timestamps, searches/upserts the customer in Airtable, creates a booking record, creates a Google Calendar event, updates the booking with the calendar event ID, sends a WhatsApp confirmation message, and returns a Flow SUCCESS screen.

**Nodes involved:**
- **Prepare Booking Data1**
- **Search Existing Customer**
- **Check Customer Exists1**
- **Upsert Customer in Airtable**
- **Lookup Service**
- **Create Booking in Airtable1**
- **Create an event**
- **Update Booking with Calendar ID1**
- **Send WhatsApp Confirmation**
- **Prepare Success Response1**
- **Merge Paths** → **Encrypt Response** → **Respond to Webhook**

#### Node: Prepare Booking Data1
- **Type / role:** Code node converting Flow form fields into normalized booking fields.
- **Key configuration choices:**
  - Parses `appointment_date` (YYYY-MM-DD) and `appointment_time` (HH:mm)
  - Duration: 60 if `consultation_type === '60_min'` else 30
  - Computes:
    - `appointmentDateTime` (ISO)
    - `appointmentEndTime` (ISO)
    - display strings for date/time (en-US locale)
  - Sets `serviceKey = consultation_type` for service lookup
  - Passes through original JSON including encryption keys and `flow_token`
- **Output →** **Search Existing Customer**
- **Edge cases:**
  - If `appointment_time` is not `HH:mm`, `split(':')` fails.
  - Using `new Date(date + 'T00:00:00')` interprets in server timezone.

#### Node: Search Existing Customer
- **Type / role:** Airtable search in Customers table.
- **Key configuration:**
  - Auth: Airtable OAuth2
  - Filter formula: `{customer_email}='{{ $json.booking.customerEmail }}'`
  - `onError: continueRegularOutput` + `alwaysOutputData: true` (workflow continues even if search fails)
- **Output →** **Check Customer Exists1**
- **Failure modes / edge cases:**
  - If email is empty, filter becomes `{customer_email}=''` which may match unintended records or none.
  - Airtable rate limits; auth expiration.

#### Node: Check Customer Exists1
- **Type / role:** Code node deciding whether to create or update customer (upsert) and building customerData.
- **Key configuration:**
  - Reads:
    - `booking` from **Prepare Booking Data1**
    - `flowToken` from **Prepare Booking Data1** `flow_token` (this is treated as phone number downstream)
    - encryption keys/version from **Prepare Booking Data1**
  - `existingCustomerId` taken from first search result `json.id` if present.
  - Creates `customerData` with:
    - `customer_phone_number: flowToken`
- **Output →** **Upsert Customer in Airtable**
- **Edge cases:**
  - If Airtable search failed but returned an error item, `existingCustomers[0].json.id` may be undefined; logic generally handles it.
  - Storing phone number in `flow_token` is a design choice: here the Flow token was set to `from` earlier in the template tool, so it effectively becomes the WhatsApp user phone.

#### Node: Upsert Customer in Airtable
- **Type / role:** Airtable upsert in Customers table.
- **Key configuration:**
  - Matching column: `id` (record id); if missing, Airtable node creates a new record.
  - Sets:
    - `customer_name`, `customer_email`
    - `date_created`, `last_interaction` to `$now`
  - **Note:** although schema includes `customer_phone_number`, the configured `columns.value` does **not** set it (it’s present in schema but not mapped).
- **Output →** **Lookup Service**
- **Edge cases:**
  - If you intended to store phone number, you must map `customer_phone_number`.
  - Using `$now` as date_created on updates may overwrite original creation timestamp.

#### Node: Lookup Service
- **Type / role:** Airtable search in Services table to find a service record by `service_key`.
- **Key configuration:**
  - Filter formula: `{service_key}='{{ $('Check Customer Exists1').first().json.booking.serviceKey }}'`
  - Requests field `service_name`
- **Output →** **Create Booking in Airtable1**
- **Edge cases:**
  - If service record not found, downstream booking may miss linkage (and currently booking creation does not map `service_type` anyway).

#### Node: Create Booking in Airtable1
- **Type / role:** Airtable create in Bookings table.
- **Key configuration:**
  - Sets:
    - `created_at = $now`
    - `event_date` and `event_time` from decrypted Flow payload (via Prepare Booking Data1 path)
    - `booking_status = "Pending"`
    - `customer_email` from upserted customer record
  - Notes mention: `service_type (linked), flow_token` etc., but **current mapping does not set `service_type` nor `flow_token`**.
- **Output →** **Create an event**
- **Edge cases:**
  - If `event_date` field type is DateTime in Airtable, passing a `YYYY-MM-DD` string may be parsed unexpectedly. (Here it uses `decrypted.data.appointment_date` directly.)
  - If customer email is blank, this may create incomplete records.

#### Node: Create an event
- **Type / role:** Google Calendar node (create event).
- **Key configuration:**
  - Calendar: `{{GOOGLE_CALENDAR_ID}}` selected in node UI
  - Start/End: from **Check Customer Exists1** booking ISO strings
  - Additional:
    - Summary: `Consultation: <customerName` (note: expression string appears to have an unmatched brace/quote in the JSON; verify in n8n UI)
    - Attendees: `{{$json.fields.customer_email}}` (coming from **Upsert Customer in Airtable** output)
    - sendUpdates: `all`
- **Output →** **Update Booking with Calendar ID1**
- **Failure modes / edge cases:**
  - If attendee email is empty/invalid, Google may reject or ignore attendee.
  - If summary expression is malformed in n8n, node will error at runtime.
  - Requires Google Calendar OAuth2 credential with event write scope.

#### Node: Update Booking with Calendar ID1
- **Type / role:** Airtable update booking record with Google Calendar event ID and set status Confirmed.
- **Key configuration:**
  - Record ID: `{{ $('Create Booking in Airtable1').item.json.id }}`
  - Sets:
    - `booking_status = "Confirmed"`
    - `calendar_event_id = {{$json.id}}` (Google event id)
- **Output →** **Send WhatsApp Confirmation**
- **Edge cases:**
  - If booking creation failed but workflow continued, record id may be missing → update fails.

#### Node: Send WhatsApp Confirmation
- **Type / role:** HTTP request to WhatsApp Graph API to send a text confirmation message.
- **Key configuration:**
  - POST `https://graph.facebook.com/v23.0/{{WHATSAPP_PHONE_NUMBER_ID}}/messages`
  - Auth: Bearer (generic credential)
  - `to`: `{{ $('Check Customer Exists1').first().json.flowToken }}`
  - Body includes booking details from **Check Customer Exists1**.
- **Output →** **Prepare Success Response1**
- **Edge cases:**
  - If `flowToken` is not actually the phone number, message will fail. (In this design it is set earlier as the WhatsApp `from`.)
  - Rate limiting or template window policies could affect deliverability (text messages generally allowed within customer care window).

#### Node: Prepare Success Response1
- **Type / role:** Code node building Flow “SUCCESS” terminal screen response and passing encryption keys.
- **Key configuration:**
  - Reads keys/version from **Check Customer Exists1**
  - Returns:
    - `screen: 'SUCCESS'`
    - `extension_message_response.params` includes `flow_token` and `booking_confirmed: true`
- **Output →** **Merge Paths**
- **Edge cases:**
  - If **Check Customer Exists1** did not run, `.first()` calls will fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GET: Verify Webhook | Webhook | Meta webhook verification endpoint (GET) | — | Verify Token | ## Utility: WhatsApp Webhook Check \nRequired for setting up your n8n webhook in Meta |
| Verify Token | Code | Validate verify token + return hub.challenge | GET: Verify Webhook | Return Challenge | ## Utility: WhatsApp Webhook Check \nRequired for setting up your n8n webhook in Meta |
| Return Challenge | Respond to Webhook | Return plaintext challenge | Verify Token | — | ## Utility: WhatsApp Webhook Check \nRequired for setting up your n8n webhook in Meta |
| POST: Receive Messages1 | Webhook | Single WhatsApp webhook receiver (POST) | — | Decrypt WhatsApp Request1 | ## Whatsapp Single Entry Point: Webhook \nBecause of Meta's Single App to Single Webhook restriction, all messages come through the same webhook (GET / POST). \nConditional statements after this are then used to filter what happens |
| Decrypt WhatsApp Request1 | Code | Parse regular messages OR decrypt Flow payload | POST: Receive Messages1 | Is Regular Message? | ## Whatsapp Single Entry Point: Webhook \nBecause of Meta's Single App to Single Webhook restriction, all messages come through the same webhook (GET / POST). \nConditional statements after this are then used to filter what happens |
| Is Regular Message? | IF | Route between regular messages vs Flow runtime | Decrypt WhatsApp Request1 | Switch; Switch on Action | ## Whatsapp Single Entry Point: Webhook \nBecause of Meta's Single App to Single Webhook restriction, all messages come through the same webhook (GET / POST). \nConditional statements after this are then used to filter what happens |
| Switch | Switch | Route regular messages by type | Is Regular Message? (true) | Return 200 OK (Regular Message); Whatsapp Agent | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| Whatsapp Agent | LangChain Agent | Decide reply vs booking template | Switch (TEXT_MESSAGE) | Return 200 OK (Regular Message) | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| OpenAI Chat Model | OpenAI Chat Model | LLM for the agent | — | Whatsapp Agent (ai_languageModel) | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| whatsapp_message_tool | HTTP Request Tool | Tool: send WhatsApp free-text message | — | Whatsapp Agent (ai_tool) | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| whatsapp_consult_template | HTTP Request Tool | Tool: send WhatsApp template that launches Flow | — | Whatsapp Agent (ai_tool) | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| Return 200 OK (Regular Message) | Respond to Webhook | Acknowledge regular message webhook | Switch; Whatsapp Agent | — | ## Intelligent Templating Assignment & Responses \nWe want to be able to fire off our template messages and flows that make sense to the customers requests. If they want more information, we can send them a link to the services website.\nIf they want to book - then a Calendar Flow, a quote, then a quote flow. |
| Switch on Action | Switch | Route decrypted Flow requests by action/screen | Is Regular Message? (false) | Return Ping Response; Handle INIT - Return Consultation Types; Prepare Calendar Request; Prepare Date Refresh Request1; Handle Confirm Booking; Prepare Booking Data1 | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Return Ping Response | Respond to Webhook | Return encrypted ping response directly | Switch on Action (PING) | — | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Handle INIT - Return Consultation Types | Code | Build SERVICE_SELECTION screen data | Switch on Action (INIT) | Merge Paths | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Prepare Calendar Request | Code | Prepare Google Events API request (today) | Switch on Action (SERVICE_SELECTION) | Google Calendar Events | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Google Calendar Events | HTTP Request | Fetch Google Calendar events for today window | Prepare Calendar Request | Calculate Available Slots | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Calculate Available Slots | Code | Compute first 8 available slots for today | Google Calendar Events | Merge Paths | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Prepare Date Refresh Request1 | Code | Prepare Google Events API request for selected date | Switch on Action (DATE_REFRESH) | Google Calendar Events (Date) | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Google Calendar Events (Date) | HTTP Request | Fetch events for selected date window | Prepare Date Refresh Request1 | Calculate Slots for Date | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Calculate Slots for Date | Code | Compute available slots for selected date | Google Calendar Events (Date) | Merge Paths | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Handle Confirm Booking | Code | Build CONFIRMATION screen content | Switch on Action (CONFIRM_BOOKING) | Merge Paths | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Prepare Booking Data1 | Code | Normalize completed booking data (times, labels) | Switch on Action (COMPLETE_BOOKING) | Search Existing Customer | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Search Existing Customer | Airtable | Search customer by email | Prepare Booking Data1 | Check Customer Exists1 | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Check Customer Exists1 | Code | Decide existing/new customer; build upsert payload | Search Existing Customer | Upsert Customer in Airtable | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Upsert Customer in Airtable | Airtable | Upsert customer record | Check Customer Exists1 | Lookup Service | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Lookup Service | Airtable | Find service record by service_key | Upsert Customer in Airtable | Create Booking in Airtable1 | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Create Booking in Airtable1 | Airtable | Create booking record | Lookup Service | Create an event | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Create an event | Google Calendar | Create calendar event for booking | Create Booking in Airtable1 | Update Booking with Calendar ID1 | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Update Booking with Calendar ID1 | Airtable | Update booking with calendar_event_id + set Confirmed | Create an event | Send WhatsApp Confirmation | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Send WhatsApp Confirmation | HTTP Request | Send confirmation WhatsApp text | Update Booking with Calendar ID1 | Prepare Success Response1 | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Prepare Success Response1 | Code | Build SUCCESS screen response | Send WhatsApp Confirmation | Merge Paths | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Encrypt Response | Code | Encrypt Flow response (AES-128-GCM) | Merge Paths | Respond to Webhook | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Respond to Webhook | Respond to Webhook | Return encrypted Flow response (text/plain) | Encrypt Response | — | ## WhatsApp Flow: Booking \nFlows are attached to messaging templates and are activated by the WhatsApp User\n\nAt each stage of the WhatsApp Flow, data is being fed to the experience. E.g Services Available, Calendar Availability, & Confirmation. On complete - a booking is made into Airtable and a Calendar Event is made |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |
| Sticky Note5 | Sticky Note | Comment block | — | — |  |

> Note: Sticky Note nodes are included as nodes but do not execute.

---

## 4. Reproducing the Workflow from Scratch

1. **Create environment variables in n8n (or equivalent secrets store)**
   - `WHATSAPP_VERIFY_TOKEN`
   - `WHATSAPP_PRIVATE_KEY` (RSA private key used to decrypt Flow requests)
   - `WHATSAPP_PRIVATE_KEY_PASSPHRASE` (if your key is encrypted)
   - `WHATSAPP_PHONE_NUMBER_ID` (Meta WhatsApp phone number id)
   - `GOOGLE_CALENDAR_ID` (optional; used by HTTP calendar calls)
   - `AIRTABLE_BASE_ID`
   - `AIRTABLE_CUSTOMERS_TABLE_ID`
   - `AIRTABLE_BOOKINGS_TABLE_ID`
   - `AIRTABLE_SERVICES_TABLE_ID`

2. **Add Webhook (GET) node**
   - Type: **Webhook**
   - Name: `GET: Verify Webhook`
   - Path: `whatsapp-webhook`
   - Method: GET
   - Response Mode: **Respond to Webhook node**

3. **Add Code node to validate Meta verification**
   - Name: `Verify Token`
   - Logic:
     - Read `hub.mode`, `hub.verify_token`, `hub.challenge` from webhook query.
     - Compare token to `$env.WHATSAPP_VERIFY_TOKEN`.
     - Output `{ challenge }`.

4. **Add Respond to Webhook node for challenge**
   - Name: `Return Challenge`
   - Respond with: **Text**
   - Body: `{{$json.challenge}}`
   - Status code: 200
   - Connect: `GET: Verify Webhook` → `Verify Token` → `Return Challenge`

5. **Add Webhook (POST) node**
   - Name: `POST: Receive Messages1`
   - Path: `whatsapp-webhook`
   - Method: POST
   - Response Mode: **Respond to Webhook node**

6. **Add Code node to parse/decrypt incoming POST**
   - Name: `Decrypt WhatsApp Request1`
   - Implement:
     - If `body.entry` exists, extract message/status and normalize into:
       - `messageType`, `from`, `text` (for text), plus `isRegularMessage: true`
     - Else decrypt Flow payload using RSA private key + AES-128-GCM.
     - For `action === 'ping'`, return `{ response: <encryptedBase64>, isRegularMessage:false }`.
     - For other Flow actions, return decrypted payload fields and `aesKeyBase64`, `ivBase64`, `version`, `isRegularMessage:false`.
   - Connect: `POST: Receive Messages1` → `Decrypt WhatsApp Request1`

7. **Add IF node**
   - Name: `Is Regular Message?`
   - Condition: Boolean equals `{{$json.isRegularMessage}}` is `true`
   - Connect: `Decrypt WhatsApp Request1` → `Is Regular Message?`

8. **Regular message branch (true): add Switch**
   - Name: `Switch`
   - Switch on: `{{$json.messageType}}`
   - Rules:
     - `text_message` → TEXT_MESSAGE output
     - (Optionally add rules for `button_reply` and `status_update` to avoid dead ends)
   - Connect: `Is Regular Message? (true)` → `Switch`

9. **Add Respond to Webhook for regular messages**
   - Name: `Return 200 OK (Regular Message)`
   - Respond JSON: `{"status":"received"}`
   - Connect:
     - `Switch` outputs for non-text → this Respond node
     - Later, also connect AI agent output → this Respond node

10. **Add OpenAI Chat Model**
    - Name: `OpenAI Chat Model`
    - Model: `gpt-4o`
    - Configure OpenAI credentials in n8n.

11. **Add two HTTP Request Tool nodes**
    - **Tool A** Name: `whatsapp_message_tool`
      - Method: POST
      - URL: `https://graph.facebook.com/v23.0/<WHATSAPP_PHONE_NUMBER_ID>/messages`
      - Auth: Bearer token credential (Meta token)
      - JSON body: send `type: "text"`, `to` = `{{$('Is Regular Message?').item.json.from}}`, and body from agent tool input.
    - **Tool B** Name: `whatsapp_consult_template`
      - Same endpoint/auth
      - JSON body: `type: "template"`, template name `personal_consultation_booking`, add Flow button component with `flow_token` set to the user’s phone.

12. **Add LangChain Agent**
    - Name: `Whatsapp Agent`
    - Input text: `{{$json.text}}`
    - System instructions: include intent classification and the rule that whatsapp_message tool must receive plain text.
    - Connect:
      - `OpenAI Chat Model` → `Whatsapp Agent` (ai_languageModel connection)
      - Tools → `Whatsapp Agent` (ai_tool connections)
      - `Switch (TEXT_MESSAGE)` → `Whatsapp Agent`
      - `Whatsapp Agent` → `Return 200 OK (Regular Message)`

13. **Flow branch (false): add Switch**
    - Name: `Switch on Action`
    - Build rules matching WhatsApp Flow payload:
      - `ping`
      - `INIT`
      - `data_exchange` at `SERVICE_SELECTION`
      - `data_exchange` at `DATE_TIME_SELECTION` with `refresh_slots`
      - `data_exchange` at `DATE_TIME_SELECTION` with `confirm_booking`
      - `data_exchange` at `CONFIRMATION` with `complete_booking`
    - Connect: `Is Regular Message? (false)` → `Switch on Action`

14. **Add Respond to Webhook for ping**
    - Name: `Return Ping Response`
    - Respond with Text: `{{$json.response}}`
    - Connect: `Switch on Action (PING)` → `Return Ping Response`

15. **Add Code node for INIT response**
    - Name: `Handle INIT - Return Consultation Types`
    - Output:
      - `response`: `{version, screen:'SERVICE_SELECTION', data:{consultation_types, min_date, max_date}}`
      - plus `aesKeyBase64`, `ivBase64`
    - Connect: `Switch on Action (INIT)` → this node

16. **Add “today availability” calendar nodes**
    - Code: `Prepare Calendar Request` (sets calendarId/timeMin/timeMax/timezone/consultationType/customer data)
    - HTTP: `Google Calendar Events` (Events list API; OAuth2 Google credential)
    - Code: `Calculate Available Slots` (creates `DATE_TIME_SELECTION` response + keys)
    - Connect: `Switch on Action (SERVICE_SELECTION)` → Prepare → HTTP → Calculate

17. **Add “selected date refresh” calendar nodes**
    - Code: `Prepare Date Refresh Request1`
    - HTTP: `Google Calendar Events (Date)`
    - Code: `Calculate Slots for Date`
    - Connect: `Switch on Action (DATE_REFRESH)` → Prepare → HTTP → Calculate

18. **Add Code node for confirm_booking**
    - Name: `Handle Confirm Booking`
    - Creates `CONFIRMATION` screen response + keys.
    - Connect: `Switch on Action (CONFIRM_BOOKING)` → this node

19. **Add booking completion chain**
    1. Code: `Prepare Booking Data1`
    2. Airtable: `Search Existing Customer` (search by `{customer_email}` equals booking email; Airtable OAuth2 credential)
    3. Code: `Check Customer Exists1` (choose existing record id or null, pass booking + flow token + keys)
    4. Airtable: `Upsert Customer in Airtable` (upsert; match `id`; set name/email/last_interaction; optionally set phone)
    5. Airtable: `Lookup Service` (search `{service_key}`)
    6. Airtable: `Create Booking in Airtable1` (create booking record)
    7. Google Calendar: `Create an event` (create event with start/end; attendees)
    8. Airtable: `Update Booking with Calendar ID1` (set calendar_event_id + booking_status Confirmed)
    9. HTTP: `Send WhatsApp Confirmation` (send text to user phone)
    10. Code: `Prepare Success Response1` (build `SUCCESS` screen response + keys)

20. **Add merge + encrypt + respond for Flow**
    - Code: `Merge Paths` (pass-through `$input.all()`)
    - Code: `Encrypt Response` (AES-128-GCM with flipped IV)
    - Respond: `Respond to Webhook` (text/plain body = encryptedResponse)
    - Connect *all Flow response-producing nodes* to `Merge Paths`:
      - `Handle INIT - Return Consultation Types`
      - `Calculate Available Slots`
      - `Calculate Slots for Date`
      - `Handle Confirm Booking`
      - `Prepare Success Response1`
    - Then `Merge Paths` → `Encrypt Response` → `Respond to Webhook`

21. **Credentials setup**
    - **Meta / WhatsApp Graph API:** create an HTTP Bearer credential with a valid access token.
    - **Google Calendar OAuth2:** connect a Google account; ensure scopes allow reading events (for HTTP nodes) and creating events (for Calendar node).
    - **Airtable OAuth2:** connect Airtable account with access to the base.

22. **WhatsApp Manager / Meta setup prerequisites**
    - Configure Webhook URL to point to your n8n webhook (GET+POST path).
    - Upload the public RSA key to WhatsApp Business “encryption” endpoint (required for Flow encryption).
    - Create template `personal_consultation_booking` with a Flow button attached.
    - Create a Flow using the JSON from the Sticky Note (and publish it).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Book meetings with Whatsapp and sync to Airtable and Google Calendar … Setup: Create a Meta Developer App + Whatsapp Business Account … upload RSA key … set up Templates and Flows … attach Flow to template” | Workflow sticky note “Sticky Note5” |
| “## WhatsApp Flow JSON Copy and paste this flow into WhatsApp Flow builder…” (contains Flow JSON defining SERVICE_SELECTION → DATE_TIME_SELECTION → CONFIRMATION) | Workflow sticky note “Sticky Note3” |
| “## Whatsapp Single Entry Point: Webhook Because of Meta's Single App to Single Webhook restriction…” | Workflow sticky note “Sticky Note” |
| “## Intelligent Templating Assignment & Responses … fire off template messages and flows that make sense …” | Workflow sticky note “Sticky Note2” |
| “## WhatsApp Flow: Booking … Calendar Availability … On complete booking synced …” | Workflow sticky note “Sticky Note1” |

