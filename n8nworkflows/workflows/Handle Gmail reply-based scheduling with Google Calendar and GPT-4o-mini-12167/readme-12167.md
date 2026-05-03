Handle Gmail reply-based scheduling with Google Calendar and GPT-4o-mini

https://n8nworkflows.xyz/workflows/handle-gmail-reply-based-scheduling-with-google-calendar-and-gpt-4o-mini-12167


# Handle Gmail reply-based scheduling with Google Calendar and GPT-4o-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Handle Gmail reply-based scheduling with Google Calendar and GPT-4o-mini  
**Workflow internal name:** Smart Calendar Availability & Auto Scheduling

**Purpose:**  
This workflow turns free-form meeting requests into structured calendar events using GPT-4o-mini, checks Google Calendar availability, creates the event if available, and (optionally) handles back-and-forth scheduling via Gmail replies to confirm or request alternative choices.

**Target use cases:**
- Auto-scheduling meetings from a message (webhook input or sample text)
- Avoiding double-bookings by checking calendar availability before creation
- Email-based “pick option 1/2” confirmation loop (optional extension)

### 1.1 Trigger & Input Reception
Receives scheduling requests via Webhook (and currently routes into a “Sample Input” node for testing).

### 1.2 AI Extraction (GPT-4o-mini) & Data Preparation
Uses an OpenAI chat model to extract event fields (JSON), then prepares a payload intended for calendar actions.

### 1.3 Availability Check & Decision
Queries Google Calendar availability and branches depending on whether the slot is free.

### 1.4 Final Action (Create vs. Alternatives)
If free: create the event. If not free: generate alternative time slot suggestions (but note: sending those alternatives is not implemented in the main path).

### 1.5 Reply Handling (Optional Extension via Gmail)
Polls Gmail for user replies, interprets whether they selected option “1”, and either creates the event + sends confirmation or asks them to reply again.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Input Reception

**Overview:**  
Starts the workflow via an HTTP webhook. In the current wiring, it immediately goes to a “Sample Input” node (so webhook payload is not used unless you rewire it).

**Nodes involved:**
- Webhook
- Sample Input

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point (HTTP)
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/ai-schedule-copilot**
- **Inputs / outputs:**
  - **Output →** Sample Input
- **Key considerations / edge cases:**
  - If you want real webhook payload usage, you must map `Webhook` body fields into later nodes (currently overridden by Sample Input).
  - Webhook security (auth header, secret, etc.) is not configured (Options empty).

#### Node: Sample Input
- **Type / role:** `n8n-nodes-base.set` — provides test data
- **Configuration (interpreted):**
  - Sets:
    - `text`: a sample meeting request containing date/time, attendee email, and agenda
    - `timezone`: `Asia/Tokyo`
- **Inputs / outputs:**
  - **Input ←** Webhook
  - **Output →** Message a model
- **Edge cases:**
  - This node *replaces* real incoming data flow unless modified. In production you’d remove it or make it conditional.

**Sticky note covering this block (applies to nodes above):**
- “## Trigger & Input  
  Trigger & input layer. Starts the workflow via webhook or test input. Provides initial data to the system.”

---

### Block 2 — AI Extraction (GPT-4o-mini) & Data Preparation

**Overview:**  
Sends the input text to GPT-4o-mini with strict instructions to return JSON describing the event. Then prepares fields used downstream (though currently the “Prepare Calendar Payload” is hard-coded and does not parse the model output).

**Nodes involved:**
- Message a model
- Prepare Calendar Payload

#### Node: Message a model
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via n8n’s LangChain integration
- **Configuration (interpreted):**
  - Model: **gpt-4o-mini**
  - System message forces **ONLY valid JSON** with schema:
    - title, start_datetime (ISO 8601), end_datetime (ISO 8601), location, attendees[], description
  - User message injects `{{ $json.text }}`
- **Credentials:** OpenAI API credential (`Kota1`)
- **Inputs / outputs:**
  - **Input ←** Sample Input
  - **Output →** Prepare Calendar Payload
- **Key expressions / variables:**
  - `{{ $json.text }}`
- **Edge cases / failure types:**
  - Model returns non-JSON or extra text (despite instruction) → downstream parsing would fail *if implemented*. (Currently no JSON parsing occurs.)
  - Rate limits / quota / invalid API key.
  - Timezone ambiguity if the source text lacks timezone; you pass `timezone` but it is not referenced in the prompt.

#### Node: Prepare Calendar Payload
- **Type / role:** `n8n-nodes-base.set` — supposed to shape event payload
- **Configuration (interpreted):**
  - Currently **hard-coded**:
    - `title`: “Collaboration meeting”
    - `start_datetime`: `2023-12-28T19:00:00+09:00`
    - `end_datetime`: `2023-12-28T20:00:00+09:00`
  - **Does not** parse or map the model’s JSON output into these fields.
- **Inputs / outputs:**
  - **Input ←** Message a model
  - **Output →** Get availability in a calendar
- **Edge cases:**
  - Since attendees/description/location are not set here, downstream nodes that reference them may resolve to `undefined` (notably “Create an event” expects attendees/description).
  - Dates are fixed; availability check and event creation will always use those fixed times unless updated.

**Sticky note covering this block:**
- “## AI & Data Preparation  
  AI processing and data preparation. Generates text and formats calendar payload. Prepares clean scheduling data.”

---

### Block 3 — Availability Check & Decision

**Overview:**  
Checks whether the calendar is available for the requested slot and branches on `available === true`.

**Nodes involved:**
- Get availability in a calendar
- If

#### Node: Get availability in a calendar
- **Type / role:** `n8n-nodes-base.googleCalendar` — Google Calendar query (availability/free-busy style)
- **Configuration (interpreted):**
  - Resource: `calendar`
  - Calendar: `user@example.com`
  - Operation appears intended as **availability check** (node name), producing an `available` boolean (used downstream).
  - **Important:** The visible parameters do not show start/end being provided; depending on the node operation defaults, this may not work as intended unless start/end are configured in the UI.
- **Credentials:** Google Calendar OAuth2 (`Google Calendar account 2`)
- **Inputs / outputs:**
  - **Input ←** Prepare Calendar Payload
  - **Output →** If
- **Edge cases / failure types:**
  - OAuth token expired/invalid scopes.
  - Missing required time window parameters (common cause of runtime errors).
  - Calendar ID mismatch (primary vs email).

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — routing decision
- **Condition (interpreted):**
  - Checks `{{$json.available}}` **is true**
- **Inputs / outputs:**
  - **Input ←** Get availability in a calendar
  - **True output →** Create an event
  - **False output →** Build Alternative Slots
- **Edge cases:**
  - If `available` is missing or not boolean, strict validation can make the condition fail (likely routes to false path).

**Sticky note covering this block:**
- “## Availability Decision  
  Availability check and decision logic. Checks if the time slot is free. Routes the workflow based on the result.”

---

### Block 4 — Final Action (Create Event vs Build Alternatives)

**Overview:**  
If available, creates an event on Google Calendar. If unavailable, builds a text list of 3 alternative slots (but does not send them in the main path).

**Nodes involved:**
- Create an event
- Build Alternative Slots

#### Node: Create an event
- **Type / role:** `n8n-nodes-base.googleCalendar` — create calendar event
- **Configuration (interpreted):**
  - Calendar: `user@example.com`
  - Start: `{{ $('Prepare Calendar Payload').item.json.start_datetime }}`
  - End: `{{ $('Prepare Calendar Payload').item.json.end_datetime }}`
  - Additional fields:
    - Summary: `{{ $('Prepare Calendar Payload').item.json.title }}`
    - Attendees: `{{ $('Prepare Calendar Payload').item.json.attendees }}`
    - Description: `{{ $('Prepare Calendar Payload').item.json.description }}`
- **Credentials:** Google Calendar OAuth2 (`Google Calendar account 2`)
- **Inputs / outputs:**
  - **Input ←** If (true branch)
  - **Output:** none connected (workflow ends here on success)
- **Edge cases / failure types:**
  - If `attendees` is not an array (or is undefined), event creation may fail or create malformed attendees.
  - The attendees field is configured as an array containing an expression plus a newline; this can lead to unexpected formatting. Prefer a clean array.
  - Start/end invalid ISO strings → API error.

#### Node: Build Alternative Slots
- **Type / role:** `n8n-nodes-base.set` — constructs fallback suggestions text
- **Configuration (interpreted):**
  - Keeps all existing fields (`include = all`, `includeOtherFields = true`)
  - Sets:
    - `requested_start` = `{{$json.start_datetime}}`
    - `requested_end` = `{{$json.end_datetime}}`
    - `alternative_slots_text` using `$moment()`:
      - Option 1: start+1 day to end+1 day
      - Option 2: start+2 days to end+2 days
      - Option 3: start+3 days to end+3 days
- **Inputs / outputs:**
  - **Input ←** If (false branch) and also from “Is Valid Confirmation” (false branch) in the optional extension
  - **Output:** not connected further in main branch (no email is sent here)
- **Edge cases:**
  - If `start_datetime`/`end_datetime` absent on the incoming item, `$moment(undefined)` will produce invalid dates.
  - Timezone handling depends on whether the datetime includes an offset (recommended).

**Sticky note covering this block:**
- “## Final Action  
  Final action layer. Creates a calendar event or sends a notification email. Completes the workflow.”  
  (Note: main path does not actually send a notification email when unavailable—only builds alternatives.)

---

### Block 5 — Reply Handling (Optional Extension via Gmail)

**Overview:**  
Separately polls Gmail for replies from a specific sender, normalizes the reply content, checks if it contains “1”, and if valid creates a calendar event and sends a confirmation. Otherwise, it replies asking the user to choose properly (and also builds alternative slots).

**Nodes involved:**
- Gmail Trigger (User Reply)
- Normalize Incoming Reply
- Check Selected Option (1 or 2)
- Prepare Confirmed Slot Data
- Prepare Alternative Request
- Is Valid Confirmation
- Create Calendar Event
- Send Confirmation Email
- Request Alternative Slots
- Build Alternative Slots (shared node from Block 4)

#### Node: Gmail Trigger (User Reply)
- **Type / role:** `n8n-nodes-base.gmailTrigger` — polling trigger for new emails
- **Configuration (interpreted):**
  - Filter: sender `user@example.com`
  - Polling: every minute
- **Credentials:** Gmail OAuth2 (`Gmail account`)
- **Outputs:**
  - **Output →** Normalize Incoming Reply
- **Edge cases / failure types:**
  - Polling triggers can miss edge cases with threads vs new messages depending on Gmail trigger behavior.
  - OAuth/scopes issues (read access required).
  - Sender filter is strict; replies from other addresses won’t be processed.

#### Node: Normalize Incoming Reply
- **Type / role:** Set — normalize fields for downstream logic
- **Configuration (interpreted):**
  - `reply_text` = `{{$json.text || $json.snippet}}`
  - `from_email` = `{{$json.from}}`
  - `thread_id` = `{{$json.threadId}}`
- **Outputs:**
  - **Output →** Check Selected Option (1 or 2)
- **Edge cases:**
  - Depending on trigger output shape, `text` may not exist; snippet may be partial.
  - `from` format often includes “Name <email>” (handled later via regex).

#### Node: Check Selected Option (1 or 2)
- **Type / role:** If — detects user selecting option 1
- **Configuration (interpreted):**
  - Condition: `reply_text` **contains** `"1"`
  - Despite the node name, it does not check for `"2"`; it only branches on presence of `"1"`.
- **Outputs:**
  - **True →** Prepare Confirmed Slot Data
  - **False →** Prepare Alternative Request
- **Edge cases:**
  - Any message containing “1” (e.g., date “Jan 10”, “11am”) will be treated as confirmation. More robust parsing is recommended (e.g., regex `^\s*1\s*$`).

#### Node: Prepare Confirmed Slot Data
- **Type / role:** Set — creates fields needed to confirm booking
- **Configuration (interpreted):**
  - Sets `selected_option = "1"` (duplicated twice)
  - Sets:
    - `chosen_start = {{$json.requested_start}}`
    - `chosen_end = {{$json.requested_end}}`
    - `attendee_email = {{$json.from_email}}`
- **Inputs / outputs:**
  - **Input ←** Check Selected Option (true)
  - **Output →** Is Valid Confirmation
- **Edge cases:**
  - `requested_start` / `requested_end` are not produced anywhere in this Gmail-trigger branch unless you store them externally (DB, static data, thread mapping). As-is, these may be undefined.
  - Duplicate field assignment (`selected_option`) is harmless but confusing.

#### Node: Prepare Alternative Request
- **Type / role:** Set — flags invalid reply
- **Configuration (interpreted):**
  - `is_valid_reply = false`
  - `reply_reason = "Invalid option (not 1)"`
- **Outputs:**
  - **Output →** Is Valid Confirmation

#### Node: Is Valid Confirmation
- **Type / role:** If — checks whether reply is valid
- **Configuration (interpreted):**
  - Condition: `is_valid_reply` is true
  - **Important:** In the “confirmed” path, `is_valid_reply` is never set to true, so this condition will likely fail and route to the “false” branch unless n8n defaults it elsewhere (it doesn’t).
- **Outputs:**
  - **True →** Create Calendar Event
  - **False →** Request Alternative Slots **and** Build Alternative Slots
- **Edge cases:**
  - Current logic likely always goes to false branch. To fix: set `is_valid_reply=true` in “Prepare Confirmed Slot Data”.

#### Node: Create Calendar Event
- **Type / role:** Google Calendar — create confirmed event
- **Configuration (interpreted):**
  - Start: `{{$json.chosen_start}}`
  - End: `{{$json.chosen_end}}`
  - Summary: “Meeting confirmed”
  - Attendees: extracts email from `attendee_email` using:
    - `{{$json.attendee_email.match(/<(.+?)>/)?.[1] || $json.attendee_email}}`
  - Description: “Scheduled via email reply”
- **Outputs:**
  - **Output →** Send Confirmation Email
- **Edge cases:**
  - If chosen_start/end are missing, Google API will fail.
  - Regex is good for `Name <email>` formats; may still fail on unusual formats.

#### Node: Send Confirmation Email
- **Type / role:** `n8n-nodes-base.gmail` — send email confirmation
- **Configuration (interpreted):**
  - To: same regex extraction as above
  - Subject: “Meeting confirmed”
  - HTML message includes chosen_start/chosen_end
- **Outputs:** none connected
- **Edge cases:**
  - Requires Gmail send scope; may fail due to restricted scopes or domain policies.
  - Uses emoji in message body; harmless but optional.

#### Node: Request Alternative Slots
- **Type / role:** Gmail — reply asking user to respond with valid options
- **Configuration (interpreted):**
  - Operation: **reply**
  - `messageId = {{$json.id}}`
  - Message: asks user to reply with “1” or “2”
- **Inputs / outputs:**
  - **Input ←** Is Valid Confirmation (false)
  - **Output:** none connected
- **Edge cases:**
  - If trigger item doesn’t include `id` in the expected field, reply will fail.
  - Replying by messageId may not keep correct threading if Gmail node expects threadId; depends on node behavior/version.

**Sticky note covering this block:**
- “## Reply Handling (Optional Extension)  
  …Checks calendar availability, sends alternative time slots if unavailable, listens for email replies, confirms and creates calendar events upon user confirmation”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point (HTTP POST) | — | Sample Input | ## Trigger & Input; Trigger & input layer. Starts the workflow via webhook or test input. Provides initial data to the system. |
| Sample Input | Set | Test payload injection | Webhook | Message a model | ## Trigger & Input; Trigger & input layer. Starts the workflow via webhook or test input. Provides initial data to the system. |
| Message a model | OpenAI (LangChain) | Extract event JSON from text | Sample Input | Prepare Calendar Payload | ## AI & Data Preparation; AI processing and data preparation. Generates text and formats calendar payload. Prepares clean scheduling data. |
| Prepare Calendar Payload | Set | Event fields for downstream calendar ops (currently hard-coded) | Message a model | Get availability in a calendar | ## AI & Data Preparation; AI processing and data preparation. Generates text and formats calendar payload. Prepares clean scheduling data. |
| Get availability in a calendar | Google Calendar | Check availability for slot | Prepare Calendar Payload | If | ## Availability Decision; Availability check and decision logic. Checks if the time slot is free. Routes the workflow based on the result. |
| If | If | Branch on availability | Get availability in a calendar | Create an event; Build Alternative Slots | ## Availability Decision; Availability check and decision logic. Checks if the time slot is free. Routes the workflow based on the result. |
| Create an event | Google Calendar | Create event (main path) | If (true) | — | ## Final Action; Final action layer. Creates a calendar event or sends a notification email. Completes the workflow. |
| Build Alternative Slots | Set | Generate 3 fallback time slots text | If (false); Is Valid Confirmation (false) | — | ## Final Action; Final action layer. Creates a calendar event or sends a notification email. Completes the workflow. |
| Gmail Trigger (User Reply) | Gmail Trigger | Poll for replies from user | — | Normalize Incoming Reply | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Normalize Incoming Reply | Set | Normalize reply fields (text/from/thread) | Gmail Trigger (User Reply) | Check Selected Option (1 or 2) | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Check Selected Option (1 or 2) | If | Detect if reply contains “1” | Normalize Incoming Reply | Prepare Confirmed Slot Data; Prepare Alternative Request | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Prepare Confirmed Slot Data | Set | Populate confirmed slot + attendee | Check Selected Option (1 or 2) (true) | Is Valid Confirmation | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Prepare Alternative Request | Set | Mark invalid reply | Check Selected Option (1 or 2) (false) | Is Valid Confirmation | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Is Valid Confirmation | If | Validate confirmation flag | Prepare Confirmed Slot Data; Prepare Alternative Request | Create Calendar Event; Request Alternative Slots; Build Alternative Slots | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Create Calendar Event | Google Calendar | Create event (reply-confirmed flow) | Is Valid Confirmation (true) | Send Confirmation Email | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Send Confirmation Email | Gmail | Notify attendee of confirmation | Create Calendar Event | — | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Request Alternative Slots | Gmail | Ask user to reply with valid option | Is Valid Confirmation (false) | — | ## Reply Handling (Optional Extension); This section demonstrates how email replies can be handled as an extension… |
| Sticky Note | Sticky Note | Global description | — | — | ## 📅 Smart Calendar Scheduling Workflow (content as shown) |
| Sticky Note1 | Sticky Note | Block label/comment | — | — | ## Trigger & Input (content as shown) |
| Sticky Note2 | Sticky Note | Block label/comment | — | — | ## AI & Data Preparation (content as shown) |
| Sticky Note3 | Sticky Note | Block label/comment | — | — | ## Availability Decision (content as shown) |
| Sticky Note4 | Sticky Note | Block label/comment | — | — | ## Final Action (content as shown) |
| Sticky Note5 | Sticky Note | Block label/comment | — | — | ## Reply Handling (Optional Extension) (content as shown) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Smart Calendar Availability & Auto Scheduling** (or your preferred name).
- (Optional) Add sticky notes to annotate blocks as in the original.

2) **Add Webhook trigger**
- Add node: **Webhook**
- Method: **POST**
- Path: **ai-schedule-copilot**
- Leave options default unless you need authentication.

3) **(Testing) Add Sample Input (Set)**
- Add node: **Set** named “Sample Input”
- Add fields:
  - `text` (string): your sample meeting request text
  - `timezone` (string): e.g., `Asia/Tokyo`
- Connect: **Webhook → Sample Input**
- Production variant: remove this node and use the Webhook body directly.

4) **Add OpenAI model node**
- Add node: **OpenAI (LangChain) / Message a model**
- Model: **gpt-4o-mini**
- Messages:
  - **System:** instruct to output ONLY valid JSON with keys: title, start_datetime, end_datetime, location, attendees, description
  - **User:** include the text: `{{ $json.text }}`
- Configure **OpenAI API credentials** in n8n (API key).
- Connect: **Sample Input → Message a model**

5) **Add “Prepare Calendar Payload” (Set)**
- Add node: **Set** named “Prepare Calendar Payload”
- To match the current workflow exactly, set hard-coded:
  - `title`, `start_datetime`, `end_datetime`
- Connect: **Message a model → Prepare Calendar Payload**
- Recommended improvement when recreating for real use:
  - Parse the model JSON (typically via a “JSON Parse” or “Code” node) and map the parsed fields instead of hard-coding.

6) **Add Google Calendar availability check**
- Add node: **Google Calendar** named “Get availability in a calendar”
- Resource: **Calendar**
- Choose the calendar: `user@example.com` (or your calendar)
- Configure OAuth2 credentials: **Google Calendar OAuth2**
- Ensure the operation you choose actually returns an `available` boolean (and configure required time window fields if prompted in UI).
- Connect: **Prepare Calendar Payload → Get availability in a calendar**

7) **Add decision node**
- Add node: **If**
- Condition: Boolean → `{{$json.available}}` is **true**
- Connect: **Get availability in a calendar → If**

8) **If TRUE: Create event**
- Add node: **Google Calendar** named “Create an event”
- Operation: **Create Event**
- Calendar: same as above
- Start: `{{ $('Prepare Calendar Payload').item.json.start_datetime }}`
- End: `{{ $('Prepare Calendar Payload').item.json.end_datetime }}`
- Additional fields:
  - Summary: `{{ $('Prepare Calendar Payload').item.json.title }}`
  - Attendees: map from `attendees` (ensure it is an array)
  - Description: map from `description`
- Connect: **If (true) → Create an event**

9) **If FALSE: Build alternatives**
- Add node: **Set** named “Build Alternative Slots”
- Keep all fields (enable include other fields)
- Add:
  - `requested_start = {{$json.start_datetime}}`
  - `requested_end = {{$json.end_datetime}}`
  - `alternative_slots_text` using `$moment()` with +1/+2/+3 days formatting (as in the original)
- Connect: **If (false) → Build Alternative Slots**
- Note: To actually notify, add a Gmail/Email node after this to send `alternative_slots_text`.

### Optional Extension: Gmail reply confirmation loop

10) **Add Gmail Trigger**
- Add node: **Gmail Trigger** named “Gmail Trigger (User Reply)”
- Poll: every minute
- Filter sender: `user@example.com` (adjust)
- Configure **Gmail OAuth2** credentials
- This becomes a second entry point (independent trigger).

11) **Normalize reply**
- Add node: **Set** named “Normalize Incoming Reply”
- Add:
  - `reply_text = {{$json.text || $json.snippet}}`
  - `from_email = {{$json.from}}`
  - `thread_id = {{$json.threadId}}`
- Connect: **Gmail Trigger → Normalize Incoming Reply**

12) **Check selected option**
- Add node: **If** named “Check Selected Option (1 or 2)”
- Condition: String → `{{$json.reply_text}}` contains `"1"`
- Connect: **Normalize Incoming Reply → Check Selected Option**

13) **Confirmed path data**
- Add node: **Set** named “Prepare Confirmed Slot Data”
- Set:
  - `selected_option = "1"`
  - `chosen_start = {{$json.requested_start}}`
  - `chosen_end = {{$json.requested_end}}`
  - `attendee_email = {{$json.from_email}}`
  - **Important fix (recommended):** `is_valid_reply = true`
- Connect: **Check Selected Option (true) → Prepare Confirmed Slot Data**

14) **Invalid path data**
- Add node: **Set** named “Prepare Alternative Request”
- Set:
  - `is_valid_reply = false`
  - `reply_reason = "Invalid option (not 1)"`
- Connect: **Check Selected Option (false) → Prepare Alternative Request**

15) **Validity gate**
- Add node: **If** named “Is Valid Confirmation”
- Condition: Boolean → `{{$json.is_valid_reply}}` is true
- Connect:
  - **Prepare Confirmed Slot Data → Is Valid Confirmation**
  - **Prepare Alternative Request → Is Valid Confirmation**

16) **On valid: Create calendar event + send confirmation**
- Add node: **Google Calendar** named “Create Calendar Event”
  - Start: `{{$json.chosen_start}}`
  - End: `{{$json.chosen_end}}`
  - Attendee: `{{$json.attendee_email.match(/<(.+?)>/)?.[1] || $json.attendee_email}}`
  - Summary: “Meeting confirmed”
- Add node: **Gmail** named “Send Confirmation Email”
  - To: same regex extraction
  - Subject: “Meeting confirmed”
  - Body: include chosen_start/end
- Connect: **Is Valid Confirmation (true) → Create Calendar Event → Send Confirmation Email**

17) **On invalid: ask again (and optionally rebuild alternatives)**
- Add node: **Gmail** named “Request Alternative Slots”
  - Operation: reply
  - `messageId = {{$json.id}}`
  - Message prompting user to reply with 1 or 2
- (Optional) Connect to **Build Alternative Slots** if you want to regenerate suggestions in the same branch.
- Connect: **Is Valid Confirmation (false) → Request Alternative Slots**
- Connect (as in original): **Is Valid Confirmation (false) → Build Alternative Slots**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## 📅 Smart Calendar Scheduling Workflow … automatically checks Google Calendar availability and either creates a new event or sends a notification email if the time slot is already booked.” | Overall sticky note description embedded in workflow. |
| “Reply Handling (Optional Extension) … listens for email replies … confirms and creates calendar events upon user confirmation” | Sticky note describing the optional Gmail-based scheduling loop. |
| Main-branch gap: alternatives are generated but not emailed | In the current main path, “Build Alternative Slots” has no downstream Gmail/notification node. |
| Data-flow gap in optional reply branch: `requested_start/requested_end` not persisted | The Gmail reply trigger branch does not retrieve original requested times; you typically need storage (DB, sheet, data store) keyed by thread_id/message_id. |
| Logic bug: `is_valid_reply` never set true in “confirmed” path | Causes “Is Valid Confirmation” to route to the invalid branch unless corrected. |