Sync Fathom meeting summaries and action items with GoHighLevel contacts

https://n8nworkflows.xyz/workflows/sync-fathom-meeting-summaries-and-action-items-with-gohighlevel-contacts-13136


# Sync Fathom meeting summaries and action items with GoHighLevel contacts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync Fathom meeting summaries and action items with GoHighLevel contacts  
**Workflow name (in JSON):** Sync Fathom meeting summaries to GoHighLevel contacts  
**Purpose:** When a Fathom meeting ends, this workflow receives the meeting payload (summary, recording link, action items), identifies **external attendees**, matches them to **GoHighLevel (LeadConnector) contacts** by email, then:
- adds an **internal conversation comment** (short summary + recording link),
- creates a **detailed contact note** (full cleaned summary + recording link),
- converts **action items** into **GHL tasks** due 7 days later.

**Typical use cases**
- Automatically logging meeting outcomes to CRM contact timelines.
- Ensuring sales/support teams see a concise summary inside the conversation thread.
- Turning meeting action items into follow-up tasks without manual entry.

### Logical Blocks
**1.1 Webhook & Data Preparation**
- Receive Fathom webhook payload, normalize data, extract external attendees, prepare note/comment bodies and action items.

**1.2 Contact Matching (Per Attendee)**
- Iterate attendee-by-attendee and search GoHighLevel contacts by attendee email.

**1.3 Conversation & Notes (Per Matched Contact)**
- Find or create a conversation for the contact.
- Post an internal comment with short summary + recording link.
- Create a detailed note on the contact record.

**1.4 Task Creation (Per Action Item)**
- If action items exist, create GHL tasks (loop) tied to the matched contact.

---

## 2. Block-by-Block Analysis

### 2.1 Webhook & Data Preparation

**Overview:** Receives the Fathom webhook and transforms the payload into one item per *external attendee*, enriched with pre-formatted content for later GHL operations.

**Nodes involved**
- **Fathom Webhook**
- **Config**
- **Parse External Attendees**
- (Sticky notes: “Sync Fathom meeting summaries…”, “1. Webhook & Data Prep”)

#### Node: Fathom Webhook
- **Type / role:** `Webhook` – entry point; listens for Fathom “New meeting content ready”.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/fathom-webhook**
- **Inputs/outputs:** No input; outputs webhook payload (likely under `$json.body` depending on n8n webhook settings).
- **Version specifics:** Webhook node **typeVersion 2**.
- **Potential failures / edge cases:**
  - Fathom not configured to send to correct URL / wrong path.
  - Payload structure changes; downstream code expects keys like `calendar_invitees`, `default_summary`, `action_items`.
  - If webhook is tested vs production mode, body shape can differ (some n8n setups place data at `$json` vs `$json.body`).

#### Node: Config
- **Type / role:** `Set` – creates a normalized config object and captures incoming data.
- **Configuration (interpreted):**
  - Raw JSON output containing:
    - `ghl_location_id`: placeholder `"YOUR_LOCATION_ID_HERE"` (must be changed)
    - `webhookData`: stores a JSON-stringified copy of the incoming body: `JSON.stringify($json.body || $json)`
- **Key expressions / variables:**
  - `{{ JSON.stringify($json.body || $json) }}`
- **Connections:**
  - Input: **Fathom Webhook**
  - Output: **Parse External Attendees**
- **Version specifics:** Set node **typeVersion 3.4**.
- **Potential failures / edge cases:**
  - **Important:** `webhookData` is stored as a **string**, but the next Code node treats it like an object (`webhookData.calendar_invitees`). If not auto-parsed elsewhere, this can break. In practice, you should store it as an object (remove `JSON.stringify`) or `JSON.parse` it in the code node.

#### Node: Parse External Attendees
- **Type / role:** `Code` – transforms webhook payload into per-attendee items and constructs formatted message content.
- **Configuration (interpreted):**
  - Reads `input.webhookData || input.body || input`.
  - Filters `calendar_invitees` to only those with `is_external === true`.
  - Builds:
    - `note_body` (detailed HTML-ish text for contact note)
    - `comment_body` (short text for internal conversation comment)
    - passes through meeting metadata and `action_items`.
- **Key logic and variables:**
  - `externalAttendees = (webhookData.calendar_invitees || []).filter(invitee => invitee.is_external === true)`
  - Date formatting: `toLocaleDateString('en-GB')` → `DD/MM/YYYY`
  - Markdown cleanup:
    - Strips markdown links: `\[text](url)` → `text`
    - Extracts brief summary primarily from “## Meeting Purpose”
    - Converts markdown-ish formatting to simple HTML tags (`<b>`, `<i>`)
  - Outputs fields per attendee:
    - `attendee_email`, `attendee_name`
    - `meeting_title`, `share_url`, `meeting_date`
    - `note_body`, `comment_body`
    - `action_items` (array)
- **Connections:**
  - Input: **Config**
  - Output: **Loop Over Attendees**
- **Version specifics:** Code node **typeVersion 2**.
- **Potential failures / edge cases:**
  - If `webhookData` is a JSON string (from Config), `webhookData.calendar_invitees` will be `undefined`. Fix by parsing.
  - If there are **no external attendees**, node returns `[]`, which effectively ends the workflow (by design).
  - If `default_summary.markdown_formatted` is missing, fallback strings are used.
  - Action items assumed to be array-like; otherwise `|| []`.

---

### 2.2 Contact Matching (Per Attendee)

**Overview:** Iterates over each external attendee and searches GoHighLevel contacts by email. If no match, that attendee is skipped.

**Nodes involved**
- **Loop Over Attendees**
- **Search GHL Contact**
- **Contact Found?**
- **No Contact - Skip**
- (Sticky note: “2. Contact Matching”)

#### Node: Loop Over Attendees
- **Type / role:** `Split In Batches` – controls iteration over attendees.
- **Configuration (interpreted):**
  - Batch settings not customized (defaults). Typically defaults to batch size = 1 in many n8n versions unless set.
  - Uses the built-in second output (“Next batch”) pattern.
- **Connections:**
  - Input: **Parse External Attendees**
  - Output (loop body): **Search GHL Contact** (connected from output index 1 per JSON wiring)
  - Receives loopback from:
    - **No Contact - Skip**
    - **Loop Over Action Items** (both “done” and “continue” paths feed back)
    - **No Action Items - Skip**
- **Version specifics:** SplitInBatches **typeVersion 3**.
- **Potential failures / edge cases:**
  - Miswiring can cause infinite loops if the “done” path loops back incorrectly. Here, the loopback is intentional after each attendee’s processing.

#### Node: Search GHL Contact
- **Type / role:** `HTTP Request` – searches GHL contacts via LeadConnector API.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://services.leadconnectorhq.com/contacts/search`
  - Body JSON:
    - `locationId`: from `Config.ghl_location_id`
    - `query`: attendee email (`$json.attendee_email`)
    - `pageLimit`: 1
  - Headers:
    - `Version: 2021-07-28`
  - Auth: `highLevelOAuth2Api` predefined credential type (OAuth2 for GoHighLevel/LeadConnector)
  - Retries: enabled (`retryOnFail: true`, `maxTries: 3`, 1s wait)
- **Connections:**
  - Input: **Loop Over Attendees**
  - Output: **Contact Found?**
- **Version specifics:** HTTP node **typeVersion 4.2**.
- **Potential failures / edge cases:**
  - OAuth misconfiguration / expired token.
  - Wrong `locationId` → empty results.
  - API rate limiting or 5xx errors (retries help but may still fail).
  - Response shape assumed to include `contacts` array.

#### Node: Contact Found?
- **Type / role:** `IF` – checks whether a contact match exists.
- **Condition:**
  - `{{ $json.contacts.length }}` **>** `0`
- **Connections:**
  - True → **Search Conversation**
  - False → **No Contact - Skip**
- **Version specifics:** IF node **typeVersion 2** (strict validation enabled).
- **Potential failures / edge cases:**
  - If API response doesn’t contain `contacts`, expression can error. Consider `($json.contacts?.length || 0)` for safety.

#### Node: No Contact - Skip
- **Type / role:** `No Operation` – placeholder to keep loop logic consistent.
- **Connections:**
  - Output → **Loop Over Attendees** (to continue with next attendee)
- **Potential failures / edge cases:** None (but it influences control flow).

---

### 2.3 Conversation & Notes (Per Matched Contact)

**Overview:** For each matched contact, locates an existing conversation or creates one, posts an internal comment, then writes a detailed meeting note.

**Nodes involved**
- **Search Conversation**
- **Conversation Exists?**
- **Use Existing Conversation**
- **Create Conversation**
- **Extract New Conversation ID**
- **Create Internal Comment**
- **Create GHL Note**
- (Sticky note: “3. Conversation & Notes”)

#### Node: Search Conversation
- **Type / role:** `HTTP Request` – searches conversations by contactId and locationId.
- **Configuration (interpreted):**
  - Method: default GET (not explicitly set; query is enabled)
  - URL: `https://services.leadconnectorhq.com/conversations/search`
  - Query params:
    - `locationId`: Config value
    - `contactId`: first matched contact id: `$('Search GHL Contact').item.json.contacts[0].id`
  - Header:
    - `Version: 2021-04-15`
  - Auth: `highLevelOAuth2Api`
  - Retries: 3 tries, 1s delay
- **Connections:**
  - Input: **Contact Found?** (true branch)
  - Output: **Conversation Exists?**
- **Potential failures / edge cases:**
  - If contacts array is unexpectedly empty despite IF, expression can error.
  - Some accounts may not allow conversation search depending on scopes.

#### Node: Conversation Exists?
- **Type / role:** `IF` – decides whether to use existing conversation or create a new one.
- **Condition:**
  - `{{ $json.conversations?.length || 0 }}` **>** `0`
- **Connections:**
  - True → **Use Existing Conversation**
  - False → **Create Conversation**
- **Potential failures / edge cases:** Response shape differences (`conversations` missing) handled via optional chaining.

#### Node: Use Existing Conversation
- **Type / role:** `Code` – extracts `conversationId` from the first search result.
- **Logic:**
  - Reads `$('Search Conversation').item.json.conversations[0].id`
  - Throws error if none found (but this node only runs when IF says it exists).
- **Output:** `{ conversationId }`
- **Connections:**
  - Output → **Create Internal Comment**
- **Potential failures / edge cases:**
  - If IF condition passes but `conversations` array is empty due to race/shape mismatch, this throws.

#### Node: Create Conversation
- **Type / role:** `HTTP Request` – creates a conversation for the contact.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://services.leadconnectorhq.com/conversations/`
  - Body:
    - `locationId`
    - `contactId`
  - Header:
    - `Version: 2021-04-15`
  - Auth: `highLevelOAuth2Api`
  - Retries: enabled
- **Connections:**
  - Output → **Extract New Conversation ID**
- **Potential failures / edge cases:**
  - Permission/scopes: conversation write required.
  - API may return different shapes (`conversation.id` vs `id`).

#### Node: Extract New Conversation ID
- **Type / role:** `Code` – normalizes the create response into `{conversationId}`.
- **Logic:**
  - `conversationId = item.conversation?.id || item.id`
  - Throws if missing.
- **Connections:**
  - Output → **Create Internal Comment**
- **Potential failures / edge cases:**
  - Unexpected response schema from API changes.

#### Node: Create Internal Comment
- **Type / role:** `HTTP Request` – posts an internal comment into the conversation.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://services.leadconnectorhq.com/conversations/messages/`
  - Body:
    - `type: "InternalComment"`
    - `conversationId`: from previous code node
    - `contactId`: from search contact result
    - `message`: `Parse External Attendees.comment_body`
  - Header:
    - `Version: 2021-04-15`
  - Auth: `highLevelOAuth2Api`
  - Retries: enabled
- **Connections:**
  - Output → **Create GHL Note**
- **Potential failures / edge cases:**
  - Scope requirements include conversations/message write.
  - If message exceeds platform limits or contains unsupported markup, could fail (message is plain text here).

#### Node: Create GHL Note
- **Type / role:** `HTTP Request` – adds a note to the contact record with detailed meeting info.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://services.leadconnectorhq.com/contacts/{contactId}/notes`
  - Body:
    - `body`: `Parse External Attendees.note_body` (stringified)
  - Header:
    - `Version: 2021-07-28`
  - Auth: `highLevelOAuth2Api`
  - Retries: enabled
- **Connections:**
  - Output → **Has Action Items?**
- **Potential failures / edge cases:**
  - Notes may interpret HTML differently; body includes `<b>` tags and newline characters. Depending on GHL rendering, you may see raw tags.
  - If the contact is not writable due to permissions, request fails.

---

### 2.4 Task Creation (Per Action Item)

**Overview:** If meeting action items exist, transforms them to GHL task payloads and creates tasks with a due date of 7 days from workflow execution at 17:00.

**Nodes involved**
- **Has Action Items?**
- **Prepare Action Items**
- **Loop Over Action Items**
- **Create GHL Task**
- **No Action Items - Skip**
- (Sticky note: “4. Task Creation”)

#### Node: Has Action Items?
- **Type / role:** `IF` – checks if there are any action items.
- **Condition:**
  - `{{ $('Parse External Attendees').item.json.action_items.length }}` **>** `0`
- **Connections:**
  - True → **Prepare Action Items**
  - False → **No Action Items - Skip**
- **Potential failures / edge cases:**
  - If `action_items` is undefined, `.length` errors (though Parse sets `action_items: webhookData.action_items || []`, so typically safe).

#### Node: No Action Items - Skip
- **Type / role:** `No Operation` – ends attendee processing and returns to attendee loop.
- **Connections:**
  - Output → **Loop Over Attendees**

#### Node: Prepare Action Items
- **Type / role:** `Code` – converts action items array to one item per task.
- **Key logic:**
  - `contactId` from searched contact.
  - `actionItems` from parsed attendee item.
  - Due date:
    - `now + 7 days`
    - time set to `17:00:00.000` local time, then converted to ISO string.
  - Output per action item:
    - `contactId`
    - `title`: `item.description`
    - `completed`: `item.completed || false`
    - `dueDate`: ISO string
    - `body`: `<b>From meeting:</b> {meetingTitle}`
- **Connections:**
  - Output → **Loop Over Action Items**
- **Potential failures / edge cases:**
  - `item.description` missing → creates tasks with empty title (GHL may reject).
  - Timezone implications: `setHours(17...)` uses server timezone; ISO output may shift relative to user’s timezone expectations.

#### Node: Loop Over Action Items
- **Type / role:** `Split In Batches` – iterates over tasks to create.
- **Connections:**
  - Input: **Prepare Action Items**
  - Output (create task path): **Create GHL Task**
  - Loopback:
    - From **Create GHL Task** back into **Loop Over Action Items** (continue)
    - On completion, returns to **Loop Over Attendees** to process next attendee
- **Potential failures / edge cases:**
  - If batch size defaults unexpectedly, could create multiple tasks per API call path. Usually safe since each item becomes one HTTP request.

#### Node: Create GHL Task
- **Type / role:** `HTTP Request` – creates a task for a contact.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://services.leadconnectorhq.com/contacts/{contactId}/tasks`
  - Body:
    - `title`, `dueDate`, `completed`, `body`
  - Header:
    - `Version: 2021-07-28`
  - Auth: `highLevelOAuth2Api`
  - Retries: enabled
- **Connections:**
  - Output → **Loop Over Action Items** (to process next action item)
- **Potential failures / edge cases:**
  - Task endpoint permissions/scope.
  - Validation errors if title/body exceeds length or title missing.
  - Rate limiting on multiple action items.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / setup info |  |  | ## Sync Fathom meeting summaries to GoHighLevel contacts / How it works / Setup / Requirements (includes required scopes) |
| Sticky Note1 | Sticky Note | Block label |  |  | ## 1. Webhook & Data Prep |
| Sticky Note2 | Sticky Note | Block label |  |  | ## 2. Contact Matching |
| Sticky Note3 | Sticky Note | Block label |  |  | ## 3. Conversation & Notes |
| Sticky Note4 | Sticky Note | Block label |  |  | ## 4. Task Creation |
| Fathom Webhook | Webhook | Receives Fathom meeting payload | — | Config | (same as top sticky note) / (Webhook & Data Prep note) |
| Config | Set | Stores locationId + normalizes incoming data | Fathom Webhook | Parse External Attendees | (same as top sticky note) / (Webhook & Data Prep note) |
| Parse External Attendees | Code | Extract external attendees; build note/comment; pass action items | Config | Loop Over Attendees | (Webhook & Data Prep note) |
| Loop Over Attendees | Split In Batches | Iterate attendee-by-attendee | Parse External Attendees; No Contact - Skip; No Action Items - Skip; Loop Over Action Items | Search GHL Contact | (Contact Matching note) |
| Search GHL Contact | HTTP Request | Search contact by attendee email | Loop Over Attendees | Contact Found? | (Contact Matching note) |
| Contact Found? | IF | Branch on contact match | Search GHL Contact | Search Conversation; No Contact - Skip | (Contact Matching note) |
| No Contact - Skip | NoOp | Skip attendee when no match | Contact Found? (false) | Loop Over Attendees | (Contact Matching note) |
| Search Conversation | HTTP Request | Search conversations by contactId | Contact Found? (true) | Conversation Exists? | (Conversation & Notes note) |
| Conversation Exists? | IF | Branch on conversation existence | Search Conversation | Use Existing Conversation; Create Conversation | (Conversation & Notes note) |
| Use Existing Conversation | Code | Extract conversationId from search results | Conversation Exists? (true) | Create Internal Comment | (Conversation & Notes note) |
| Create Conversation | HTTP Request | Create conversation for contact | Conversation Exists? (false) | Extract New Conversation ID | (Conversation & Notes note) |
| Extract New Conversation ID | Code | Extract conversationId from create response | Create Conversation | Create Internal Comment | (Conversation & Notes note) |
| Create Internal Comment | HTTP Request | Post internal comment with brief summary + recording | Use Existing Conversation; Extract New Conversation ID | Create GHL Note | (Conversation & Notes note) |
| Create GHL Note | HTTP Request | Create detailed contact note | Create Internal Comment | Has Action Items? | (Conversation & Notes note) |
| Has Action Items? | IF | Decide whether to create tasks | Create GHL Note | Prepare Action Items; No Action Items - Skip | (Task Creation note) |
| No Action Items - Skip | NoOp | Skip task creation | Has Action Items? (false) | Loop Over Attendees | (Task Creation note) |
| Prepare Action Items | Code | Map action items to task payloads + due date | Has Action Items? (true) | Loop Over Action Items | (Task Creation note) |
| Loop Over Action Items | Split In Batches | Iterate tasks | Prepare Action Items; Create GHL Task | Create GHL Task; Loop Over Attendees | (Task Creation note) |
| Create GHL Task | HTTP Request | Create a task in GHL for the contact | Loop Over Action Items | Loop Over Action Items | (Task Creation note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Sync Fathom meeting summaries to GoHighLevel contacts* (or your preferred name)

2) **Add Webhook node**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: `fathom-webhook`
   - Save the workflow to generate a test URL / production URL.

3) **Add Set node (“Config”)**
   - Mode: **Raw JSON**
   - Fields to output (as JSON):
     - `ghl_location_id`: set to your GoHighLevel **Location ID**
     - `webhookData`: store the incoming webhook payload
   - Recommended (to avoid string/object mismatch): set `webhookData` as an object, not a string:
     - Use `{{$json.body || $json}}` (no JSON.stringify)
   - Connect: **Webhook → Config**

4) **Add Code node (“Parse External Attendees”)**
   - Paste logic that:
     - filters `calendar_invitees` where `is_external === true`
     - builds:
       - `comment_body` (short summary)
       - `note_body` (detailed summary)
     - returns one item per attendee with fields:
       - `attendee_email`, `attendee_name`, `meeting_title`, `share_url`, `meeting_date`, `comment_body`, `note_body`, `action_items`
   - If you kept `webhookData` as a string, add `JSON.parse(input.webhookData)` here.
   - Connect: **Config → Parse External Attendees**

5) **Add Split In Batches (“Loop Over Attendees”)**
   - Batch size: typically **1** (set explicitly to avoid ambiguity).
   - Connect: **Parse External Attendees → Loop Over Attendees**

6) **Add HTTP Request (“Search GHL Contact”)**
   - Auth: **GoHighLevel OAuth2** credential (predefined credential type in n8n: `highLevelOAuth2Api`)
     - Ensure scopes include at minimum those in the sticky note:  
       `contacts.readonly`, `contacts.write`, `conversations.readonly`, `conversations.write`, `conversations/message.write`
   - Method: **POST**
   - URL: `https://services.leadconnectorhq.com/contacts/search`
   - Headers: `Version: 2021-07-28`
   - Body (JSON):
     - `locationId`: from Config node
     - `query`: `{{$json.attendee_email}}`
     - `pageLimit`: 1
   - Retries: 3, wait 1000ms (optional but recommended)
   - Connect: **Loop Over Attendees (loop output) → Search GHL Contact**

7) **Add IF (“Contact Found?”)**
   - Condition: `{{$json.contacts.length}} > 0` (or safer: `{{$json.contacts?.length || 0}} > 0`)
   - True → continue processing
   - False → skip attendee
   - Connect: **Search GHL Contact → Contact Found?**

8) **Add NoOp (“No Contact - Skip”)**
   - Connect: **Contact Found? (false) → No Contact - Skip → Loop Over Attendees** (to continue loop)

9) **Add HTTP Request (“Search Conversation”)**
   - Method: GET (default)
   - URL: `https://services.leadconnectorhq.com/conversations/search`
   - Query params:
     - `locationId`: `{{$('Config').item.json.ghl_location_id}}`
     - `contactId`: `{{$('Search GHL Contact').item.json.contacts[0].id}}`
   - Header: `Version: 2021-04-15`
   - Auth: same GHL OAuth2 credential
   - Connect: **Contact Found? (true) → Search Conversation**

10) **Add IF (“Conversation Exists?”)**
   - Condition: `{{$json.conversations?.length || 0}} > 0`
   - True → extract existing conversationId
   - False → create new conversation
   - Connect: **Search Conversation → Conversation Exists?**

11) **Add Code (“Use Existing Conversation”)**
   - Extract `conversations[0].id` and output `{ conversationId }`
   - Connect: **Conversation Exists? (true) → Use Existing Conversation**

12) **Add HTTP Request (“Create Conversation”)**
   - Method: **POST**
   - URL: `https://services.leadconnectorhq.com/conversations/`
   - Header: `Version: 2021-04-15`
   - Body:
     - `locationId`
     - `contactId`
   - Connect: **Conversation Exists? (false) → Create Conversation**

13) **Add Code (“Extract New Conversation ID”)**
   - Output `{ conversationId: item.conversation?.id || item.id }`
   - Connect: **Create Conversation → Extract New Conversation ID**

14) **Add HTTP Request (“Create Internal Comment”)**
   - Method: **POST**
   - URL: `https://services.leadconnectorhq.com/conversations/messages/`
   - Header: `Version: 2021-04-15`
   - Body:
     - `type`: `InternalComment`
     - `conversationId`: from whichever branch produced it
     - `contactId`: from contact search
     - `message`: from parsed attendee `comment_body`
   - Connect both branches:
     - **Use Existing Conversation → Create Internal Comment**
     - **Extract New Conversation ID → Create Internal Comment**

15) **Add HTTP Request (“Create GHL Note”)**
   - Method: **POST**
   - URL: `https://services.leadconnectorhq.com/contacts/{contactId}/notes`
   - Header: `Version: 2021-07-28`
   - Body: `{ "body": note_body }`
   - Connect: **Create Internal Comment → Create GHL Note**

16) **Add IF (“Has Action Items?”)**
   - Condition: `{{$('Parse External Attendees').item.json.action_items.length}} > 0`
   - True → prepare tasks
   - False → skip
   - Connect: **Create GHL Note → Has Action Items?**

17) **Add NoOp (“No Action Items - Skip”)**
   - Connect: **Has Action Items? (false) → No Action Items - Skip → Loop Over Attendees**

18) **Add Code (“Prepare Action Items”)**
   - Map each action item to:
     - `contactId`, `title`, `completed`, `dueDate` (now + 7 days at 17:00), `body`
   - Connect: **Has Action Items? (true) → Prepare Action Items**

19) **Add Split In Batches (“Loop Over Action Items”)**
   - Batch size: 1 recommended
   - Connect: **Prepare Action Items → Loop Over Action Items**

20) **Add HTTP Request (“Create GHL Task”)**
   - Method: **POST**
   - URL: `https://services.leadconnectorhq.com/contacts/{contactId}/tasks`
   - Header: `Version: 2021-07-28`
   - Body: `title`, `dueDate`, `completed`, `body`
   - Connect: **Loop Over Action Items → Create GHL Task → Loop Over Action Items** (to continue loop)

21) **Complete the loop back to attendees**
   - Connect the “done”/completion output of **Loop Over Action Items** back to **Loop Over Attendees** (so after finishing tasks for an attendee, the workflow moves to the next attendee).

22) **Credentials & external setup**
   - In n8n: configure **GoHighLevel OAuth2** credential for HTTP Request nodes (`highLevelOAuth2Api`).
   - In Fathom: set webhook target to the **production** webhook URL:
     - Settings → API → Webhooks → “New meeting content ready” → paste URL.

23) **Activate workflow**
   - Switch workflow to **Active**.
   - Send a real meeting completion event from Fathom to validate end-to-end behavior.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Copy the webhook URL to Fathom: Settings → API → Webhooks → “New meeting content ready” | Fathom webhook configuration |
| Required GHL scopes: `contacts.readonly`, `contacts.write`, `conversations.readonly`, `conversations.write`, `conversations/message.write` | GoHighLevel/LeadConnector OAuth permissions |
| Set your `ghl_location_id` in the **Config** node | Workflow configuration requirement |
| Connect GoHighLevel OAuth2 credentials to the HTTP Request nodes | n8n credential setup requirement |

