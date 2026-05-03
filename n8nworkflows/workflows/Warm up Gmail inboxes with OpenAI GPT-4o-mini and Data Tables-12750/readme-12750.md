Warm up Gmail inboxes with OpenAI GPT-4o-mini and Data Tables

https://n8nworkflows.xyz/workflows/warm-up-gmail-inboxes-with-openai-gpt-4o-mini-and-data-tables-12750


# Warm up Gmail inboxes with OpenAI GPT-4o-mini and Data Tables

## 1. Workflow Overview

**Title:** Warm up Gmail inboxes with OpenAI GPT-4o-mini and Data Tables  
**Purpose:** Automate “inbox warmup” by generating natural, friendly, non-sales email conversations (2–3 messages), planning who talks to whom each day, sending messages at randomized times within a safe daily window, and then marking resulting warmup emails as read to mimic human behavior.

**Primary use cases**
- Gradually warming multiple Gmail inboxes by creating realistic inter-inbox conversation activity.
- Controlling daily volume per inbox and distributing conversations across inbox pairs.
- Running fully on schedules using n8n Data Tables as storage/queueing.

### 1.1 Logical Blocks
1. **Generate warmup conversations (daily)**  
   Generates conversation JSON using OpenAI and stores it in `warmup_conversations`.
2. **Build warmup queue (daily)**  
   For each sender inbox, selects random recipient inboxes, assigns conversations, expands them into scheduled message rows, and stores them in `warmup_queue`.
3. **Send warmup messages (hourly)**  
   For today’s queue items, assigns randomized send timestamps (IST window), sends new threads or replies in-thread using the sender’s Gmail credential, labels sent mail, and updates the queue row as sent with thread/message IDs.
4. **Mark warmup emails as read (hourly)**  
   Finds sent messages that should already have been delivered, fetches unread warmup messages in the receiver inbox, labels them, removes the `UNREAD` label, and updates `marked_read` in the queue.

---

## 2. Block-by-Block Analysis

### Block 1 — Generate warmup conversations (daily)
**Overview:** Runs once daily to generate multiple “conversation snippets” (each 2–3 messages) via OpenAI and stores them as JSON strings in a Data Table for later assignment.  
**Nodes involved:**  
- `generate warmup conversations`  
- `Get warmup accounts (cold_email_accounts)`  
- `Build conversation generation loop`  
- `Loop: generate conversations`  
- `Generate conversations (OpenAI)`  
- `Format conversations for Data Table`  
- `Save conversations (warmup_conversations)`

#### Node: `generate warmup conversations`
- **Type / role:** Schedule Trigger; entry point for daily generation.
- **Config:** Triggers daily at **06:00**.
- **Outputs:** Kicks off chain to load accounts and compute a loop count.
- **Failure modes:** n8n schedule disabled/timezone mismatch.

#### Node: `Get warmup accounts (cold_email_accounts)`
- **Type / role:** Data Table (get); reads account configuration.
- **Config:** `get` + `returnAll: true` from Data Table **cold_email_accounts**.
- **Key fields expected downstream:** `warmup_daily_limit` is read from the *first* returned row.
- **Failure modes / edge cases:**
  - Empty table ⇒ `$input.first()` in next node fails.
  - If `warmup_daily_limit` differs per account, only the first row is used (may be unintended).

#### Node: `Build conversation generation loop`
- **Type / role:** Code; creates N dummy items to drive generation loop.
- **Config choices:** Reads `daily_limit = $input.first().json.warmup_daily_limit` and returns `[{index:1}, ...]`.
- **Output:** N items for generation.
- **Edge cases:** Missing/non-numeric `warmup_daily_limit` ⇒ loop count invalid.

#### Node: `Loop: generate conversations`
- **Type / role:** Split In Batches; iterator to call OpenAI per item.
- **Config:** Default batch behavior (batch size not explicitly set).
- **Connections:** Its loop output feeds OpenAI; completion loops back from save step.
- **Failure modes:** If not wired correctly, may process only one batch.

#### Node: `Generate conversations (OpenAI)`
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi`; generates JSON conversation content.
- **Model:** `gpt-4o-mini`
- **Output formatting:** Enforced `json_object`.
- **Prompt behavior:**
  - Must output **ONLY JSON** with a `conversations` array of 2–3 alternating messages.
  - Additional “system” entries include example arrays; these are unusual as separate system messages and may bias output or cause format inconsistencies.
- **Credentials:** OpenAI credential named `OpenAi account`.
- **Failure modes / edge cases:**
  - Model returns invalid JSON or wrong schema ⇒ parsing/format node fails.
  - Rate limits/timeouts.
  - The examples shown are arrays, while the required format is an object with `conversations:`—could lead to occasional schema drift.

#### Node: `Format conversations for Data Table`
- **Type / role:** Code; extracts `conversations` from the OpenAI response and stringifies it.
- **Key expression:** `item.json.output[0].content[0].text.conversations`
- **Output:** Adds `conversation_string` = JSON string.
- **Edge cases / failure modes:**
  - If OpenAI node output shape differs (common across versions), this path breaks.
  - If model returns different keys, `conversations` undefined.

#### Node: `Save conversations (warmup_conversations)`
- **Type / role:** Data Table (insert/upsert style mapping); stores conversation strings.
- **Config:** Maps column `messages` to `{{$json.conversation_string}}` into table **warmup_conversations**.
- **Matching:** `matchingColumns: ["messages"]` (acts like dedupe by exact message string).
- **Failure modes:** Data Table schema mismatch; large strings; duplicates if minor changes.

---

### Block 2 — Build warmup queue (daily)
**Overview:** Runs daily, loops through each warmup account, selects random peer inboxes, takes conversation templates, expands them into per-day message rows, and inserts rows into `warmup_queue`.  
**Nodes involved:**  
- `Daily trigger: build warmup queue`  
- `Get warmup accounts for queue`  
- `Loop: build queue per inbox`  
- `Calculate daily warmup target`  
- `Get recipient inboxes (excluding sender)`  
- `Pick random recipient inboxes`  
- `Get conversations from table`  
- `Parse conversation JSON`  
- `Merge recipients + conversations`  
- `Loop: match conversations to recipients`  
- `Split conversation into scheduled messages`  
- `Flatten message list`  
- `Insert messages into warmup_queue`

#### Node: `Daily trigger: build warmup queue`
- **Type / role:** Schedule Trigger; entry point for queue creation.
- **Config:** Daily at **06:25**.
- **Failure modes:** Schedule disabled/timezone mismatch.

#### Node: `Get warmup accounts for queue`
- **Type / role:** Data Table (get).
- **Config:** Reads **cold_email_accounts**.
- **Failure modes:** Empty accounts ⇒ no queue.

#### Node: `Loop: build queue per inbox`
- **Type / role:** Split In Batches; iterates per sender inbox.
- **Connections:** Drives `Calculate daily warmup target` and later receives expanded conversation rows back (via `Split conversation...`).
- **Edge cases:** If iteration is miswired, you can end up mixing results across senders.

#### Node: `Calculate daily warmup target`
- **Type / role:** Code; computes `warmup_target_today`.
- **Logic:**  
  - `growth = 2`, `max = 40`  
  - `todayCount = min($json.warmup_sent_today + growth, max)`
- **Output:** Adds `warmup_target_today`.
- **Important edge case:** `warmup_sent_today` is referenced but not created in this workflow. If it’s missing, JS yields `undefined + 2` ⇒ `NaN`, which breaks downstream selection and batch size.
- **Note:** `base = 5` is declared but never used.

#### Node: `Get recipient inboxes (excluding sender)`
- **Type / role:** Data Table (get with filter).
- **Config:** Pulls from **cold_email_accounts** where `email != {{$json.email}}`.
- **Failure modes:** Only one inbox in table ⇒ empty recipients.

#### Node: `Pick random recipient inboxes`
- **Type / role:** Code; shuffles recipients and selects `target`.
- **Inputs:**  
  - Sender and `warmup_target_today` pulled from `Calculate daily warmup target` node output.
  - Recipient list from `Get recipient inboxes...`
- **Output:** Items shaped as `{ inboxA: sender, inboxB: recipientEmail, target }`
- **Edge cases:**  
  - If `target` is `NaN` or > recipients length, results are wrong or empty.
  - No guard to prevent pairing the same recipient across days.

#### Node: `Get conversations from table`
- **Type / role:** Data Table (get).
- **Config:** `limit = {{$json.warmup_daily_limit}}` from the sender account row, reads from **warmup_conversations**.
- **Edge cases:** If there are fewer conversations than limit, you’ll get fewer rows.

#### Node: `Parse conversation JSON`
- **Type / role:** Code; parses stored JSON strings from `warmup_conversations`.
- **Logic:** `JSON.parse(items[i].json.messages || "[]")` and returns objects:  
  `{ conversation_id: <row.id>, message: <array> }`
- **Failure modes:** Invalid JSON in `messages` column.

#### Node: `Merge recipients + conversations`
- **Type / role:** Merge (combine).
- **Config:** `combineByPosition`
- **Meaning:** Recipient item #1 pairs with conversation item #1, etc.
- **Edge cases:** If counts differ, some recipients or conversations are dropped or mispaired depending on n8n merge behavior.

#### Node: `Loop: match conversations to recipients`
- **Type / role:** Split In Batches; controls bundling.
- **Config:** `batchSize = {{$json.target}}`, with `reset: true`.
- **Edge cases:** If `target` invalid, batching breaks.

#### Node: `Split conversation into scheduled messages`
- **Type / role:** Code (runOnceForEachItem); expands a conversation into multiple queue rows.
- **Logic highlights:**
  - Alternates sender/receiver based on `message[i].sender === "A"`.
  - `scheduled_date = today + i days` (date string only).
  - Sets `scheduled_at: null` for later randomization.
  - Sets `sent: false`.
- **Output:** `{ conversation: [ ...rows ] }`
- **Edge cases:**
  - If the conversation message array is empty or malformed, produces no rows.
  - Timezone: `scheduled_date` uses the server date at execution time (not explicitly IST).

#### Node: `Flatten message list`
- **Type / role:** Code; flattens nested conversation arrays into one list of message rows.
- **Output:** One item per warmup email to insert into queue.

#### Node: `Insert messages into warmup_queue`
- **Type / role:** Data Table (insert).
- **Config:** Inserts fields: `sender_inbox`, `receiver_inbox`, `subject`, `body`, `scheduled_date`, `sent`, `marked_read=false`, `conversation_id`, `conversation_index`.
- **Schema expectations:** `warmup_queue` contains many operational fields (scheduled_at, message/thread ids, etc.).
- **Edge cases:** No dedupe logic: daily runs can create duplicates if not cleaned.

---

### Block 3 — Send warmup messages (hourly)
**Overview:** Every hour, loads today’s unsent queue items; assigns a randomized timestamp if missing; sends any that are due (scheduled_at <= now). First message starts a new thread; later messages reply in-thread using stored receiver thread IDs; updates the queue row as sent and labels messages.  
**Nodes involved:**  
- `Hourly trigger: send scheduled warmup emails`  
- `Get today unsent warmups`  
- `If scheduled_at is empty`  
- `Assign random send time (IST window)`  
- `Update scheduled_at in queue`  
- `Filter messages due now`  
- `Get accounts (for credentials)`  
- `Merge queue with account credentials`  
- `Loop: send warmup message`  
- `If first message in thread`  
- `Send Gmail message (new thread)`  
- `Get previous message (thread context)`  
- `Reply in existing Gmail thread`  
- `Label sent warmup email`  
- `Update queue: sent + message/thread IDs`

#### Node: `Hourly trigger: send scheduled warmup emails`
- **Type / role:** Schedule Trigger.
- **Config:** Every hour.
- **Notes:** Runs in parallel with credential table fetch.

#### Node: `Get today unsent warmups`
- **Type / role:** Data Table (get).
- **Filter:**  
  - `scheduled_date == {{$now.format('yyyy-MM-dd')}}`
  - `sent isFalse`
- **Edge cases:** If `scheduled_date` uses different timezone than `$now`, items may drift.

#### Node: `If scheduled_at is empty`
- **Type / role:** IF; decides whether to assign send times.
- **Condition:** `scheduled_at` is empty.
- **True branch:** assign & update; **False branch:** filter due now directly.

#### Node: `Assign random send time (IST window)`
- **Type / role:** Code; assigns random `scheduled_at` within IST 08:00–22:00.
- **Implementation detail:** Creates a “today at X” local Date, then subtracts 5.5h to produce UTC ISO.
- **Edge cases:**
  - Server timezone differences can make this conversion inaccurate (it assumes the constructed Date is “IST-like” but actually uses server local time).
  - If `scheduled_at` already exists, it is preserved.

#### Node: `Update scheduled_at in queue`
- **Type / role:** Data Table (update).
- **Filter:** Updates by `id = {{$json.id}}` (provided as a filter condition with only `keyValue`; relies on n8n default key being `id`—ensure this is correct in your environment).
- **Updates:** `scheduled_at`, forces `sent=false`, `marked_read=false`.
- **Failure modes:** Misconfigured filter could update the wrong row or none.

#### Node: `Filter messages due now`
- **Type / role:** Code; selects rows where `scheduled_at <= now`.
- **Output:** Due items only.

#### Node: `Get accounts (for credentials)`
- **Type / role:** Data Table (get all).
- **Purpose:** Provide `cred_id` for each sender inbox.
- **Edge cases:** Missing `cred_id` causes send nodes to fail.

#### Node: `Merge queue with account credentials`
- **Type / role:** Merge (by fields).
- **Config:** Advanced merge on `sender_inbox` (queue) == `email` (accounts), prefer queue values on clashes.
- **Output:** Each due queue row enriched with `cred_id` (and any other account fields).
- **Failure modes:** No matching account ⇒ row lacks credential ID.

#### Node: `Loop: send warmup message`
- **Type / role:** Split In Batches; iterates per due message.
- **Output path:** Second output goes to `If first message in thread`.

#### Node: `If first message in thread`
- **Type / role:** IF; decides new thread vs reply.
- **Condition:** `conversation_index == "0"` (string compare).
- **Edge cases:** If `conversation_index` is numeric 0, it may still stringify; but if it’s stored differently, the comparison may fail.

#### Node: `Send Gmail message (new thread)`
- **Type / role:** Run Node With Credentials (dynamic); sends Gmail message using credential ID from the row.
- **Inner node:** Gmail “Send a message” with:
  - To: `receiver_inbox`
  - Subject/body from row
  - `appendAttribution: false`
- **Credentials:** `credentialsId = {{$json.cred_id}}`
- **Failure modes:** Invalid credential ID; Gmail quota; invalid “from” permission.

#### Node: `Get previous message (thread context)`
- **Type / role:** Data Table (get); tries to locate previous queue row by `id - 1`.
- **Purpose:** Obtain prior message/thread IDs for replies.
- **Major edge case:** Data Table `id` is not guaranteed to be sequential per conversation; `id - 1` may refer to a different conversation or sender. This is brittle.

#### Node: `Reply in existing Gmail thread`
- **Type / role:** Run Node With Credentials (dynamic); Gmail thread reply.
- **Inner config:**
  - `resource: thread`, `operation: reply`
  - `threadId = {{$json.receiver_thread_id}}`
  - `messageId = {{$json.receiver_thread_id}}` (likely wrong; messageId should be a message ID, not thread ID)
  - `message = {{$('If first message in thread').item.json.body}}`
  - `onError: continueErrorOutput`
- **Credentials:** `{{$('If first message in thread').item.json.cred_id}}`
- **Failure modes / edge cases:**
  - Missing `receiver_thread_id` (not set yet) ⇒ reply fails.
  - Incorrect `messageId` mapping can fail Gmail API.
  - Thread ownership: replying from the sender account to a thread that exists in the receiver inbox may not work unless IDs correspond in that mailbox context.

#### Node: `Label sent warmup email`
- **Type / role:** Run Node With Credentials; adds labels to the sent message.
- **Labels:** `IMPORTANT`, `Label_1`
- **Edge cases:** `Label_1` must exist in each Gmail account or Gmail will error.

#### Node: `Update queue: sent + message/thread IDs`
- **Type / role:** Data Table (update); stores Gmail IDs and marks row as sent.
- **Filter:** Updates row by `id = {{$('Loop: send warmup message').item.json.id}}`.
- **Updates:** `sent=true`, `thread_id`, `message_id`, `marked_read=false`.
- **Edge cases:** If Gmail node returns different field names, mapping breaks (expects `threadId` and `id`).

---

### Block 4 — Mark warmup emails as read (hourly)
**Overview:** Every hour (at minute 25), finds sent warmups that should already be delivered, then in the receiver inbox fetches unread messages from the sender, labels them, removes `UNREAD`, and updates the queue row with receiver-side message/thread IDs and `marked_read=true`.  
**Nodes involved:**  
- `Hourly trigger: mark warmups as read`  
- `Get sent warmups to mark read`  
- `Get accounts (for receiver creds)`  
- `Merge receiver inbox credentials`  
- `Loop: mark warmups read`  
- `Fetch unread messages from sender`  
- `If warmup row exists`  
- `Label warmup message (receiver)`  
- `Remove UNREAD label (mark read)`  
- `Update queue: marked_read = true`

#### Node: `Hourly trigger: mark warmups as read`
- **Type / role:** Schedule Trigger.
- **Config:** Every hour at **minute 25**.

#### Node: `Get sent warmups to mark read`
- **Type / role:** Data Table (get).
- **Filters (all):**
  - `sent isTrue`
  - `marked_read isFalse`
  - `scheduled_at < {{ DateTime.now().setZone('Asia/Kolkata').toISO() }}`
- **alwaysOutputData:** true (prevents workflow from stopping when no rows).
- **Edge cases:** If `scheduled_at` is stored UTC but compared against IST ISO string, lexicographic/temporal comparisons depend on Data Table’s comparison semantics.

#### Node: `Get accounts (for receiver creds)`
- **Type / role:** Data Table (get).
- **Purpose:** Provide `cred_id` for receiver inboxes.

#### Node: `Merge receiver inbox credentials`
- **Type / role:** Merge (by fields).
- **Config:** Join by `receiver_inbox` (queue) == `email` (accounts).
- **Output:** Queue row enriched with receiver’s `cred_id`.
- **Failure modes:** Missing receiver in accounts table.

#### Node: `Loop: mark warmups read`
- **Type / role:** Split In Batches; iterates each warmup row to mark read.
- **Wiring:** Its second output feeds `Fetch unread messages from sender`, and workflow loops back after updating queue.

#### Node: `Fetch unread messages from sender`
- **Type / role:** Run Node With Credentials; Gmail “getAll” messages (unread) filtered by sender.
- **Filter:** `readStatus: unread`, `sender: {{$json.sender_inbox}}`
- **Credentials:** `credentialsId = {{$json.cred_id}}` (receiver’s credential after merge).
- **Edge cases:**
  - Fetches *all* unread messages from that sender, not necessarily the specific warmup message/thread.
  - Could mark unrelated real emails as read if sender_inbox overlaps with genuine mail.

#### Node: `If warmup row exists`
- **Type / role:** IF; checks `$json` not empty.
- **Behavior:** True => label & mark read; False => loops.

#### Node: `Label warmup message (receiver)`
- **Type / role:** Run Node With Credentials; adds labels on the receiver side.
- **Credentials expression:** `{{$('Loop Over Items5').item.json.cred_id}}`
- **Major issue:** Node references **`Loop Over Items5`**, which does not exist in this workflow JSON. This will fail at runtime unless that node exists or was renamed without updating expressions. It likely should reference `Loop: mark warmups read`.
- **Labels:** `IMPORTANT`, `Label_1`
- **Failure modes:** broken expression; label missing.

#### Node: `Remove UNREAD label (mark read)`
- **Type / role:** Run Node With Credentials; removes `UNREAD` label.
- **Credentials expression:** same invalid `Loop Over Items5` reference.
- **Failure modes:** broken expression; message already read.

#### Node: `Update queue: marked_read = true`
- **Type / role:** Data Table (update).
- **Filter:** `id = {{$('Loop: mark warmups read').item.json.id}}`
- **Updates:** `marked_read=true`, and stores `receiver_thread_id`, `receiver_message_id` from Gmail fetch result (`threadId`, `id`).
- **Edge cases:** If multiple unread messages are returned, mapping may update queue with IDs of the wrong message (depends on which item is flowing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Block label / documentation | — | — | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Sticky Note1 | Sticky Note | Block label / documentation | — | — | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Sticky Note2 | Sticky Note | Block label / documentation | — | — | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Sticky Note3 | Sticky Note | Block label / documentation | — | — | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Sticky Note5 | Sticky Note | Global workflow notes | — | — | ## How it works\nThis workflow warms up Gmail inboxes by sending natural, non-sales email conversations between your own inboxes. It generates friendly multi-message conversations using OpenAI, stores them in `warmup_conversations`, builds a daily send plan in `warmup_queue`, and assigns randomized send times within a safe daily window. Messages are sent using the correct Gmail OAuth credential for each inbox, replies happen in the same thread, and warmup emails are automatically marked as read to mimic human inbox activity.\n\n## Setup steps\n1) Create 3 Data Tables: `cold_email_accounts`, `warmup_conversations`, `warmup_queue`\n2) Add at least 2 inboxes to `cold_email_accounts` with `email`, `cred_id`, and `warmup_daily_limit`\n3) Create one Gmail OAuth2 credential per inbox and paste the credential ID into `cred_id`\n4) Add your OpenAI API credential to the OpenAI node\n5) Enable schedule triggers and execute once to test |
| generate warmup conversations | Schedule Trigger | Daily entry: generate conversation templates | — | Get warmup accounts (cold_email_accounts) | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Get warmup accounts (cold_email_accounts) | Data Table | Load inbox/account config for generation | generate warmup conversations | Build conversation generation loop | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Build conversation generation loop | Code | Create N items from warmup_daily_limit | Get warmup accounts (cold_email_accounts) | Loop: generate conversations | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Loop: generate conversations | Split In Batches | Iterate OpenAI generation N times | Build conversation generation loop / Save conversations (warmup_conversations) | Generate conversations (OpenAI) | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Generate conversations (OpenAI) | OpenAI (LangChain) | Generate JSON conversation | Loop: generate conversations | Format conversations for Data Table | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Format conversations for Data Table | Code | Extract + stringify conversations | Generate conversations (OpenAI) | Save conversations (warmup_conversations) | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Save conversations (warmup_conversations) | Data Table | Store conversation templates | Format conversations for Data Table | Loop: generate conversations | ## 1) Generate warmup conversations (daily)\nCreates friendly conversation JSON and saves it to `warmup_conversations`. |
| Daily trigger: build warmup queue | Schedule Trigger | Daily entry: build queue | — | Get warmup accounts for queue | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Get warmup accounts for queue | Data Table | Load inboxes for queue building | Daily trigger: build warmup queue | Loop: build queue per inbox | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Loop: build queue per inbox | Split In Batches | Iterate per sender inbox | Get warmup accounts for queue / Split conversation into scheduled messages | Calculate daily warmup target; Flatten message list | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Calculate daily warmup target | Code | Compute warmup_target_today per sender | Loop: build queue per inbox | Get recipient inboxes (excluding sender); Get conversations from table | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Get recipient inboxes (excluding sender) | Data Table | List possible recipients excluding sender | Calculate daily warmup target | Pick random recipient inboxes | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Pick random recipient inboxes | Code | Shuffle/select recipients to meet target | Get recipient inboxes (excluding sender) | Merge recipients + conversations | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Get conversations from table | Data Table | Fetch conversation templates for assignment | Calculate daily warmup target | Parse conversation JSON | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Parse conversation JSON | Code | Parse stored JSON into message arrays | Get conversations from table | Merge recipients + conversations | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Merge recipients + conversations | Merge | Pair recipient items with conversation items | Parse conversation JSON; Pick random recipient inboxes | Loop: match conversations to recipients | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Loop: match conversations to recipients | Split In Batches | Batch pairing/processing | Merge recipients + conversations | Split conversation into scheduled messages | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Split conversation into scheduled messages | Code | Expand conversation into dated queue rows | Loop: match conversations to recipients | Loop: build queue per inbox | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Flatten message list | Code | Flatten expanded rows for insertion | Loop: build queue per inbox | Insert messages into warmup_queue | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Insert messages into warmup_queue | Data Table | Persist message rows to send later | Flatten message list | — | ## 2) Build warmup queue (daily)\nPairs inboxes + assigns messages into `warmup_queue`. |
| Hourly trigger: send scheduled warmup emails | Schedule Trigger | Hourly entry: send due emails | — | Get today unsent warmups; Get accounts (for credentials) | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Get today unsent warmups | Data Table | Fetch today’s unsent queue rows | Hourly trigger: send scheduled warmup emails | If scheduled_at is empty | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| If scheduled_at is empty | IF | Branch: assign send times if missing | Get today unsent warmups | Assign random send time (IST window); Filter messages due now | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Assign random send time (IST window) | Code | Set scheduled_at randomly for each row | If scheduled_at is empty (true) | Update scheduled_at in queue | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Update scheduled_at in queue | Data Table | Persist scheduled_at back to queue | Assign random send time (IST window) | Filter messages due now | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Filter messages due now | Code | Keep only rows due at/after now | If scheduled_at is empty (false) / Update scheduled_at in queue | Merge queue with account credentials | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Get accounts (for credentials) | Data Table | Load sender inbox cred_id mapping | Hourly trigger: send scheduled warmup emails | Merge queue with account credentials | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Merge queue with account credentials | Merge | Join due rows to sender credential | Filter messages due now; Get accounts (for credentials) | Loop: send warmup message | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Loop: send warmup message | Split In Batches | Iterate sending each due message | Merge queue with account credentials / Update queue: sent + message/thread IDs | If first message in thread | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| If first message in thread | IF | Choose new thread vs reply | Loop: send warmup message | Send Gmail message (new thread); Get previous message (thread context) | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Send Gmail message (new thread) | Run With Credentials | Gmail send (new thread) using sender cred | If first message in thread (true) | Label sent warmup email | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Get previous message (thread context) | Data Table | Fetch prior queue row (id-1) for reply context | If first message in thread (false) | Reply in existing Gmail thread | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Reply in existing Gmail thread | Run With Credentials | Gmail thread reply using sender cred | Get previous message (thread context) | Label sent warmup email | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Label sent warmup email | Run With Credentials | Apply labels to sent message | Send Gmail message (new thread) / Reply in existing Gmail thread | Update queue: sent + message/thread IDs | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Update queue: sent + message/thread IDs | Data Table | Store Gmail IDs, sent=true | Label sent warmup email | Loop: send warmup message | ## 3) Send warmup messages (hourly)\nAssigns random send times and sends/replies via Gmail. |
| Hourly trigger: mark warmups as read | Schedule Trigger | Hourly entry: mark receiver side read | — | Get sent warmups to mark read; Get accounts (for receiver creds) | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Get sent warmups to mark read | Data Table | Fetch sent, unmarked_read rows | Hourly trigger: mark warmups as read | Merge receiver inbox credentials | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Get accounts (for receiver creds) | Data Table | Load receiver inbox cred_id mapping | Hourly trigger: mark warmups as read | Merge receiver inbox credentials | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Merge receiver inbox credentials | Merge | Join queue rows to receiver credential | Get sent warmups to mark read; Get accounts (for receiver creds) | Loop: mark warmups read | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Loop: mark warmups read | Split In Batches | Iterate marking read per row | Merge receiver inbox credentials / Update queue: marked_read = true | Fetch unread messages from sender | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Fetch unread messages from sender | Run With Credentials | Fetch unread emails in receiver inbox from sender | Loop: mark warmups read | If warmup row exists | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| If warmup row exists | IF | Guard: only proceed if message exists | Fetch unread messages from sender | Label warmup message (receiver) / Loop: mark warmups read | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Label warmup message (receiver) | Run With Credentials | Apply labels in receiver inbox | If warmup row exists (true) | Remove UNREAD label (mark read) | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Remove UNREAD label (mark read) | Run With Credentials | Remove UNREAD label in receiver inbox | Label warmup message (receiver) | Update queue: marked_read = true | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |
| Update queue: marked_read = true | Data Table | Persist marked_read + receiver IDs | Remove UNREAD label (mark read) | Loop: mark warmups read | ## 4) Mark warmup emails as read (hourly)\nRemoves UNREAD and updates `marked_read` in queue. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Data Tables**
1. Create Data Table **`cold_email_accounts`** with (minimum) columns:
   - `email` (string)
   - `cred_id` (string) — the n8n credential ID for that inbox’s Gmail OAuth2
   - `warmup_daily_limit` (number/string)
   - `warmup_sent_today` (number) *(required by current logic; otherwise fix the code)*
2. Create Data Table **`warmup_conversations`** with column:
   - `messages` (string) — stores JSON string of the conversation message array
3. Create Data Table **`warmup_queue`** with columns (as used by workflow):
   - `sender_inbox` (string), `receiver_inbox` (string)
   - `subject` (string), `body` (string)
   - `scheduled_date` (string `YYYY-MM-DD`)
   - `scheduled_at` (dateTime)
   - `sent` (boolean)
   - `message_id`, `thread_id` (string)
   - `receiver_message_id`, `receiver_thread_id` (string)
   - `marked_read` (boolean)
   - `conversation_id` (string)
   - `conversation_index` (string/number)
   - (optional) `status` (string) if you want to keep parity with the update schema

2) **Create credentials**
4. Create one **Gmail OAuth2** credential per inbox to be warmed.
5. Copy each credential’s internal **credential ID** and store it in `cold_email_accounts.cred_id` for that inbox.
6. Create an **OpenAI API** credential and keep it available for the OpenAI node.

3) **Block 1: Generate warmup conversations**
7. Add **Schedule Trigger** node named `generate warmup conversations`, set time to daily **06:00**.
8. Add **Data Table** node `Get warmup accounts (cold_email_accounts)` (operation: Get, return all) pointing to `cold_email_accounts`.
9. Add **Code** node `Build conversation generation loop` that reads `$input.first().json.warmup_daily_limit` and returns N dummy items.
10. Add **Split In Batches** node `Loop: generate conversations`, connect from the Code node.
11. Add **OpenAI (LangChain)** node `Generate conversations (OpenAI)`:
    - Model: `gpt-4o-mini`
    - Force JSON output (json_object)
    - System prompt: same constraints (2–3 messages; A/B; standalone emails; no links/signatures/sales/spammy terms; output JSON with `conversations` array)
    - Attach OpenAI credential.
12. Add **Code** node `Format conversations for Data Table` to extract the OpenAI response into a JSON string field `conversation_string`.
13. Add **Data Table** node `Save conversations (warmup_conversations)` to insert into `warmup_conversations.messages = {{$json.conversation_string}}`.
14. Connect `Save conversations...` back to `Loop: generate conversations` to continue batching.

4) **Block 2: Build warmup queue**
15. Add **Schedule Trigger** `Daily trigger: build warmup queue` at **06:25**.
16. Add **Data Table** `Get warmup accounts for queue` (Get) from `cold_email_accounts`.
17. Add **Split In Batches** `Loop: build queue per inbox`.
18. Add **Code** `Calculate daily warmup target` to compute `warmup_target_today`.  
    - Either ensure `warmup_sent_today` exists, or modify code to default it (e.g., `($json.warmup_sent_today ?? base)`).
19. Add **Data Table** `Get recipient inboxes (excluding sender)` filtering `email != {{$json.email}}`.
20. Add **Code** `Pick random recipient inboxes` to shuffle and pick `target` recipients; output items with `inboxA`, `inboxB`, `target`.
21. Add **Data Table** `Get conversations from table` from `warmup_conversations` with limit `{{$json.warmup_daily_limit}}`.
22. Add **Code** `Parse conversation JSON` to parse each row `messages` string into arrays and output `{conversation_id, message}`.
23. Add **Merge** `Merge recipients + conversations` in `combineByPosition` mode:
    - Input 1: parsed conversations
    - Input 2: selected recipient pairs
24. Add **Split In Batches** `Loop: match conversations to recipients` with `batchSize = {{$json.target}}`, `reset: true`.
25. Add **Code** `Split conversation into scheduled messages` (runOnceForEachItem) to expand each conversation into per-day rows.
26. Connect `Split conversation...` back into `Loop: build queue per inbox` (as in the original design).
27. Add **Code** `Flatten message list` connected from `Loop: build queue per inbox` to flatten all expanded rows.
28. Add **Data Table** `Insert messages into warmup_queue` to insert each message row (sent=false, marked_read=false, scheduled_at null).

5) **Block 3: Send warmup messages (hourly)**
29. Add **Schedule Trigger** `Hourly trigger: send scheduled warmup emails` set to every **1 hour**.
30. Add **Data Table** `Get today unsent warmups` from `warmup_queue` with filters:
    - `scheduled_date == {{$now.format('yyyy-MM-dd')}}`
    - `sent isFalse`
31. Add **IF** node `If scheduled_at is empty` checking `scheduled_at` empty.
32. True branch: add **Code** `Assign random send time (IST window)` to set `scheduled_at` randomly (08:00–22:00 IST) if missing.
33. Add **Data Table** `Update scheduled_at in queue` (operation update) filtered by the row `id`, updating `scheduled_at`.
34. From both paths, add **Code** `Filter messages due now` to keep `scheduled_at <= now`.
35. Add **Data Table** `Get accounts (for credentials)` to get all rows from `cold_email_accounts`.
36. Add **Merge** `Merge queue with account credentials` merging by `sender_inbox == email` to attach `cred_id`.
37. Add **Split In Batches** `Loop: send warmup message` to process each due row.
38. Add **IF** `If first message in thread` where `conversation_index == "0"`.
39. True branch: add **Run Node With Credentials** node `Send Gmail message (new thread)` that wraps a Gmail “Send” node; set dynamic `credentialsId = {{$json.cred_id}}`.
40. False branch: add **Data Table** `Get previous message (thread context)` (note: original uses `id - 1`, but you should instead fetch by `conversation_id` + `conversation_index - 1` + matching inbox pair to be reliable).
41. Add **Run Node With Credentials** node `Reply in existing Gmail thread` that wraps Gmail “Thread → Reply”; use sender credentials dynamically.
42. After both send/reply paths, add **Run Node With Credentials** `Label sent warmup email` to apply labels (`IMPORTANT`, `Label_1`).
43. Add **Data Table** `Update queue: sent + message/thread IDs` to set `sent=true` and store returned `message_id/thread_id`.

6) **Block 4: Mark warmups as read (hourly)**
44. Add **Schedule Trigger** `Hourly trigger: mark warmups as read` every hour at minute **25**.
45. Add **Data Table** `Get sent warmups to mark read` with filters:
    - `sent isTrue`
    - `marked_read isFalse`
    - `scheduled_at < now (IST)`
46. Add **Data Table** `Get accounts (for receiver creds)` from `cold_email_accounts`.
47. Add **Merge** `Merge receiver inbox credentials` by `receiver_inbox == email` to attach receiver `cred_id`.
48. Add **Split In Batches** `Loop: mark warmups read`.
49. Add **Run Node With Credentials** `Fetch unread messages from sender` to fetch unread messages in receiver inbox where sender is `sender_inbox` (credentialsId = receiver cred_id).
50. Add **IF** `If warmup row exists` to ensure some message was fetched.
51. Add **Run Node With Credentials** `Label warmup message (receiver)` to label the fetched message, then `Remove UNREAD label (mark read)` to remove `UNREAD`.
52. Add **Data Table** `Update queue: marked_read = true` to set `marked_read=true` and store receiver thread/message IDs.
53. **Fix required:** Ensure the credentials expressions in the receiver labeling/removal nodes reference an existing loop node (replace the broken `$('Loop Over Items5')...` with `$('Loop: mark warmups read')...` or use `$json.cred_id` after merge).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer provided by user |
| “How it works” + “Setup steps” text (see Sticky Note5) | Embedded workflow notes (Sticky Note5) |

