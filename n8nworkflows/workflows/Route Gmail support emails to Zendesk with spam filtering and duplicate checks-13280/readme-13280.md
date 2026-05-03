Route Gmail support emails to Zendesk with spam filtering and duplicate checks

https://n8nworkflows.xyz/workflows/route-gmail-support-emails-to-zendesk-with-spam-filtering-and-duplicate-checks-13280


# Route Gmail support emails to Zendesk with spam filtering and duplicate checks

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically convert incoming Gmail support emails into **Zendesk tickets**, while keeping Zendesk clean by **blocking spam early** and **preventing duplicate tickets** (by comparing subjects against existing tickets).

**Typical use cases**
- Small/medium support teams receiving requests in a Gmail inbox and managing them in Zendesk.
- Reducing Zendesk noise from spam and repeated emails.
- Adding lightweight categorization via tags (refund/damaged/late delivery).

### 1.1 Input Reception (Gmail polling)
Starts the workflow when a new email arrives.

### 1.2 Spam Detection & Early Stop
Applies keyword/domain heuristics; if spam is detected, the execution ends immediately.

### 1.3 Data Preparation & Tag Enrichment
Normalizes fields from the Gmail payload into a stable structure and adds tags derived from the email snippet/content.

### 1.4 Zendesk Context Fetch + Merge
Fetches existing Zendesk tickets, then merges them with the prepared email for comparison.

### 1.5 Duplicate Detection & Validation Gate
Checks whether an existing ticket has the same normalized subject; if so, stops. Otherwise, allows ticket creation.

### 1.6 Ticket Creation in Zendesk
Creates a Zendesk ticket with requester info, subject, comment body, tags, and external thread identifier.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Gmail polling)
**Overview:** Watches the support Gmail inbox and triggers every minute to pull new messages.

**Nodes involved**
- New Support Email Trigger

#### Node: **New Support Email Trigger**
- **Type / role:** `gmailTrigger` — entry point; polls Gmail for new emails.
- **Key configuration (interpreted):**
  - Polling schedule: **every minute**
  - Filters: none configured (empty object), so it relies on default trigger behavior (typically “new email” in the chosen mailbox/label depending on node defaults).
- **Credentials:** Gmail OAuth2 required (configured in workflow but placeholder ids/names in JSON).
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** Gmail message metadata (includes keys like `From`, `Subject`, `snippet`, `threadId`, `internalDate`, etc. depending on node settings/defaults).
- **Version-specific:** node typeVersion **1.3**.
- **Edge cases / failures:**
  - OAuth token expiry / revoked consent.
  - Polling rate limits from Gmail.
  - Missing fields if the trigger returns a different shape than expected by downstream expressions (e.g., `From` absent).

**Sticky note (applies to this block’s nodes)**
- “Incoming Support Email Trigger … listens for new customer messages … starts … in real time.”

---

### Block 2 — Spam Detection & Early Filtering
**Overview:** Applies simple heuristics (keywords + disposable email domains). Spam ends the workflow by returning no items.

**Nodes involved**
- Spam Detection & Filtering

#### Node: **Spam Detection & Filtering**
- **Type / role:** `code` — custom JavaScript filtering.
- **Key configuration choices:**
  - Reads (lowercased): `subject`, `description`, `customer_email` from the current item.
  - Checks for:
    - **Spam keywords** in subject/body: `win money`, `free gift`, `click here`, `urgent action`, `lottery`, `bitcoin`, `crypto`, `investment opportunity`
    - **Disposable domains** in sender email: `tempmail`, `10minutemail`, `mailinator`, `guerrillamail`
  - If spam is detected: `return []` (drops the item and ends downstream path).
- **Expressions/variables:**
  - Uses `$json` as `item`
  - Relies on `item.subject`, `item.description`, `item.customer_email`
- **Inputs / outputs:**
  - **Input:** Output of Gmail trigger (raw Gmail fields)
  - **Output:** Either the original item (if not spam) or **no output items** (spam).
- **Version-specific:** typeVersion **2**.
- **Edge cases / failures (important):**
  - **Field mismatch:** At this point in the workflow, the Gmail trigger output likely uses keys like `Subject`, `From`, `snippet`, not `subject/description/customer_email`. Because the code defaults missing values to `''`, spam detection may become ineffective (false negatives) unless upstream fields already match.
  - Over-filtering risk if legitimate emails contain keywords (e.g., “crypto” in a legitimate request).
  - Disposable domain detection uses `email.includes(d)` which will match partial strings (acceptable but can false-positive if unusual addresses contain those substrings).

**Sticky note**
- “Spam Detection & Early Filtering … checks subject, content and sender domain … stopped immediately…”

---

### Block 3 — Prepare Email Data & Tag Enrichment
**Overview:** Converts Gmail payload fields into normalized fields used throughout the workflow (customer name/email, subject, description/snippet, thread id). Then adds more tags based on content.

**Nodes involved**
- Extract & Normalize Email Data
- Auto-Tag Ticket Based on Email Content

#### Node: **Extract & Normalize Email Data**
- **Type / role:** `set` — maps and derives normalized fields.
- **Key configuration choices:**
  - Creates/overwrites:
    - `customer_name` = everything before `" <"` in `From`
    - `customer_email` = regex capture inside `<...>` from `From`
    - `subject` = `Subject`
    - `description` = `snippet`
    - `thread_id` = `threadId`
    - `received_at` = `internalDate`
    - `tags` = if snippet contains “refund” → `["refund"]` else `[]`
- **Key expressions / variables used:**
  - `{{$json["From"].split(" <")[0]}}`
  - `{{$json["From"].match(/<(.+)>/)[1]}}`
  - `{{$json["Subject"]}}`
  - `{{$json["snippet"]}}`
  - `{{ $json.threadId }}`
  - `{{$json["internalDate"]}}`
  - `{{$json["snippet"].toLowerCase().includes("refund") ? ["refund"] : []}}`
- **Inputs / outputs:**
  - **Input:** From Spam Detection node
  - **Output:** One item with normalized fields; used in parallel by:
    - Auto-tag code node
    - Zendesk “getAll tickets”
- **Version-specific:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If `From` is not in `Name <email@domain>` format, `split(" <")` or `.match(/<(.+)>/)` can throw or return null, causing expression evaluation errors.
  - `internalDate` may be a string timestamp; later consumers may expect ISO date.
  - Initial `tags` is set as an **array**, but later Zendesk node sets `"tags": "{{ $json.tags }}"` which may serialize unexpectedly depending on Zendesk node behavior.

#### Node: **Auto-Tag Ticket Based on Email Content**
- **Type / role:** `code` — enriches tags based on description content.
- **Key configuration choices:**
  - Reads `description`, lowercases it.
  - Keyword → tag mapping:
    - `refund` → `refund`
    - `damaged` → `damaged`
    - `late delivery` → `late_delivery`
  - Produces **new `tags` array** (note: overwrites any existing `tags` from the Set node).
- **Inputs / outputs:**
  - **Input:** normalized email item
  - **Output:** normalized email item with `tags` (array)
- **Version-specific:** typeVersion **2**.
- **Edge cases / failures:**
  - If `description` is missing/null, `.toLowerCase()` will throw.
  - Overwrites the earlier `tags` field entirely; if you intend to keep pre-tags (like the Set node’s refund check), you must merge arrays instead of replacing.

**Sticky note**
- “Prepare Email Data & Check Existing Tickets … extracts customer details … retrieves existing Zendesk tickets … accurate comparison…”

---

### Block 4 — Zendesk Context Fetch + Merge
**Overview:** Pulls all Zendesk tickets, then merges them with the single incoming email item to enable duplicate checks.

**Nodes involved**
- Zendesk – Fetch Existing Tickets
- Merge Email With Existing Tickets

#### Node: **Zendesk – Fetch Existing Tickets**
- **Type / role:** `zendesk` — retrieves tickets for comparison.
- **Key configuration choices:**
  - Operation: **Get All**
  - Authentication: **OAuth2**
  - Options: default/empty
- **Inputs / outputs:**
  - **Input:** receives the email item (only to keep execution context; the node itself queries Zendesk).
  - **Output:** multiple items (one per ticket) representing existing tickets.
- **Version-specific:** typeVersion **1**.
- **Edge cases / failures:**
  - OAuth2 misconfiguration, token expiry.
  - Pagination/large account: “getAll” may be slow or hit API limits; may time out on very large ticket volumes.
  - Zendesk permissions: integration user may not access all tickets.

#### Node: **Merge Email With Existing Tickets**
- **Type / role:** `merge` — combines two inputs: (0) email item, (1) Zendesk tickets.
- **Key configuration choices:**
  - Uses default merge behavior (not explicitly configured).
- **Inputs / outputs:**
  - **Input 0:** from Auto-Tag Ticket Based on Email Content (single email item)
  - **Input 1:** from Zendesk – Fetch Existing Tickets (many ticket items)
  - **Output:** a merged stream consumed by duplicate detection code node.
- **Version-specific:** typeVersion **3.2**.
- **Edge cases / failures:**
  - With default merge mode, output ordering/shape can be non-obvious. The next code node assumes **item[0] is email** and the rest are tickets—this is not guaranteed unless the merge mode preserves that ordering consistently.
  - If either input is empty, the output can differ from expectations.

**Sticky note**
- Same as Block 3 note (“Prepare Email Data & Check Existing Tickets …”).

---

### Block 5 — Duplicate Detection & Validation Gate
**Overview:** Uses subject normalization to detect duplicates. If duplicate, ends. If not, passes the email forward into an IF gate (which is currently misconfigured).

**Nodes involved**
- Detect Duplicate Ticket by Subject
- Is This a New Ticket?

#### Node: **Detect Duplicate Ticket by Subject**
- **Type / role:** `code` — compares incoming email subject to Zendesk ticket subjects.
- **Key configuration choices:**
  - Reads all incoming items: `const items = $input.all()`
  - Assumes:
    - `items[0]` is the **email**
    - `items[1..]` are Zendesk tickets
  - Normalizes via `trim().toLowerCase()`
  - If any ticket subject equals email subject → returns `[]` (stop).
  - Else returns `[ { json: email } ]` (continue).
  - `alwaysOutputData: true` is enabled (node-level), which affects behavior in certain failure/empty-output scenarios in n8n UI/debugging.
- **Inputs / outputs:**
  - **Input:** merged items from Merge node
  - **Output:** either no items (duplicate) or one item (email)
- **Version-specific:** typeVersion **2**.
- **Edge cases / failures:**
  - If merge output order is different, the comparison becomes invalid (could treat a Zendesk ticket as “email”).
  - Throws hard error if `email.subject` missing: `throw new Error('Missing email subject')`.
  - Only checks **subject**, not thread id / requester / message-id. Legitimate separate issues with same subject could be incorrectly blocked.
  - Zendesk ticket payload field may not be `ticket.subject` depending on node output shape; if different, duplicates won’t be detected.

#### Node: **Is This a New Ticket?**
- **Type / role:** `if` — intended as a gate before creating a ticket.
- **Key configuration choices (as implemented):**
  - Condition: `{{$json.isEmpty()}}` is **true**
- **Connections / behavior:**
  - **True branch (index 0):** not connected
  - **False branch (index 1):** goes to “Create Zendesk Support Ticket”
- **Critical issue (logic flaw):**
  - `$json` is a plain object and does not have an `isEmpty()` function in standard n8n expressions. This expression will typically evaluate to an error or undefined, making the IF unreliable.
  - Even if it worked, the workflow currently creates tickets on the **false** branch, which is counterintuitive.
  - Practically, duplicate prevention already happens in the previous code node by returning `[]`; this IF node is unnecessary unless reworked.
- **Version-specific:** typeVersion **2.2**.
- **Edge cases / failures:**
  - Expression evaluation error halting execution.
  - Misrouted branch causing tickets not to be created or created unexpectedly.

**Sticky note**
- “Duplicate Detection & Validation … compared with existing Zendesk tickets … only valid new tickets move forward.”

---

### Block 6 — Zendesk Ticket Creation
**Overview:** Creates a Zendesk ticket from the normalized email fields.

**Nodes involved**
- Create Zendesk Support Ticket

#### Node: **Create Zendesk Support Ticket**
- **Type / role:** `zendesk` — creates a new ticket.
- **Key configuration choices:**
  - Authentication: OAuth2 (credential named “Zendesk account 2”)
  - Uses **JSON parameters** with a constructed payload:
    - `subject`: from `$json.subject`
    - `comment.body`: from `$json.description`
    - `requester.name`: from `$json.customer_name`
    - `requester.email`: from `$json.customer_email`
    - `tags`: from `$json.tags`
    - `priority`: `"high"`
    - `status`: `"open"`
    - `external_id`: from `$json.thread_id` (Gmail thread id)
- **Inputs / outputs:**
  - **Input:** email item (post-duplicate-check, and post-IF gate)
  - **Output:** created ticket response from Zendesk API.
- **Version-specific:** typeVersion **1**.
- **Edge cases / failures:**
  - Zendesk validation errors if requester email invalid, tags wrong type, or required fields missing.
  - `tags` may need to be an array of strings; using `"{{ $json.tags }}"` inside a JSON string can convert arrays to strings depending on how the node interpolates (test in your n8n/Zendesk node version).
  - If `external_id` already exists and Zendesk account enforces uniqueness in a certain way, create may fail or behave unexpectedly.

**Sticky note**
- “Zendesk Ticket Creation … creates a new support ticket … includes customer details … priority, tags and external identifiers…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Support Email Trigger | Gmail Trigger | Entry point; polls Gmail for new support emails | — | Spam Detection & Filtering | Incoming Support Email Trigger: listens for new customer messages; starts automation flow in real time. |
| Spam Detection & Filtering | Code | Heuristic spam filter; drops spam by outputting no items | New Support Email Trigger | Extract & Normalize Email Data | Spam Detection & Early Filtering: checks subject/content/sender domain; stops suspicious emails immediately. |
| Extract & Normalize Email Data | Set | Extracts customer name/email, subject, snippet, thread id; initial tags | Spam Detection & Filtering | Auto-Tag Ticket Based on Email Content; Zendesk – Fetch Existing Tickets | Prepare Email Data & Check Existing Tickets: structures email data; retrieves Zendesk tickets for comparison. |
| Auto-Tag Ticket Based on Email Content | Code | Adds tags based on description keywords | Extract & Normalize Email Data | Merge Email With Existing Tickets | Prepare Email Data & Check Existing Tickets: structures email data; retrieves Zendesk tickets for comparison. |
| Zendesk – Fetch Existing Tickets | Zendesk | Retrieves existing tickets (get all) for duplicate detection | Extract & Normalize Email Data | Merge Email With Existing Tickets | Prepare Email Data & Check Existing Tickets: structures email data; retrieves Zendesk tickets for comparison. |
| Merge Email With Existing Tickets | Merge | Combines email item and ticket list into one stream | Auto-Tag Ticket Based on Email Content; Zendesk – Fetch Existing Tickets | Detect Duplicate Ticket by Subject | Duplicate Detection & Validation: compares incoming email with existing tickets; prevents duplicates. |
| Detect Duplicate Ticket by Subject | Code | Subject-based duplicate check; stops if match found | Merge Email With Existing Tickets | Is This a New Ticket? | Duplicate Detection & Validation: compares incoming email with existing tickets; prevents duplicates. |
| Is This a New Ticket? | IF | Gate before creation (currently misconfigured) | Detect Duplicate Ticket by Subject | Create Zendesk Support Ticket (false branch) | Duplicate Detection & Validation: compares incoming email with existing tickets; prevents duplicates. |
| Create Zendesk Support Ticket | Zendesk | Creates Zendesk ticket with requester/comment/tags/external_id | Is This a New Ticket? | — | Zendesk Ticket Creation: creates a new ticket with complete actionable request details. |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 | Sticky Note | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Gmail Support Emails to Zendesk Tickets with Spam & Duplicate Prevention* (or your desired title).

2) **Add trigger: Gmail**
- Add node: **Gmail Trigger** named **“New Support Email Trigger”**
- Configure:
  - Polling: **Every minute**
  - Choose the Gmail account (OAuth2) that receives support emails
  - Optional but recommended: set label/mailbox filters for your support inbox/label
- Connect it to the next step.

3) **Add spam filter (Code)**
- Add node: **Code** named **“Spam Detection & Filtering”**
- Paste JS logic that:
  - Reads the relevant fields
  - Checks spam keywords and disposable domains
  - `return []` if spam; otherwise returns the item
- **Important adjustment for correctness:** Ensure the code reads Gmail fields as they exist at this point (commonly `Subject`, `From`, `snippet`). Alternatively, move spam detection **after** the Set node so it can rely on `subject/description/customer_email`.

4) **Add normalization (Set)**
- Add node: **Set** named **“Extract & Normalize Email Data”**
- Add fields (string unless noted):
  - `customer_name` = parse from `From` (name part)
  - `customer_email` = parse from `From` (email inside `< >`)
  - `subject` = Gmail `Subject`
  - `description` = Gmail `snippet` (or full body if you fetch it)
  - `thread_id` = Gmail `threadId`
  - `received_at` = Gmail `internalDate`
  - `tags` = expression returning an **array** (e.g., `["refund"]` or `[]`)
- Connect Spam Filter → Set.

5) **Add auto-tagging (Code)**
- Add node: **Code** named **“Auto-Tag Ticket Based on Email Content”**
- Implement keyword detection on `description` and output `tags` as an array.
- Connect Set → Auto-Tag.

6) **Add Zendesk fetch (existing tickets)**
- Add node: **Zendesk** named **“Zendesk – Fetch Existing Tickets”**
- Authentication: **OAuth2**
- Operation: **Get All** (tickets)
- Connect Set → Zendesk Fetch (this runs in parallel to auto-tag).

7) **Merge email + tickets**
- Add node: **Merge** named **“Merge Email With Existing Tickets”**
- Connect:
  - Auto-Tag → Merge **Input 1** (or Input 0; just be consistent)
  - Zendesk Fetch → Merge **Input 2**
- Configure merge mode explicitly so downstream logic is deterministic (recommended). For example:
  - Use a mode that keeps the email item identifiable (or merge by position with a guaranteed ordering).

8) **Add duplicate detection (Code)**
- Add node: **Code** named **“Detect Duplicate Ticket by Subject”**
- Logic:
  - Identify which incoming item is the email vs. tickets (avoid relying on position if possible).
  - Normalize subjects (`trim().toLowerCase()`)
  - If duplicate found → `return []`
  - Else → return email item
- Connect Merge → Duplicate Check.

9) **(Optional) Fix or remove the IF gate**
- Current workflow includes an IF named **“Is This a New Ticket?”**, but it is misconfigured.
- Recommended reproduction options:
  - **Option A (simplest):** Remove the IF node and connect Duplicate Check → Create Ticket.
  - **Option B (keep IF):** Set IF condition to something valid, e.g. “subject exists” or a boolean field like `isDuplicate` produced by the code node, then route only “not duplicate” to ticket creation.

10) **Create Zendesk ticket**
- Add node: **Zendesk** named **“Create Zendesk Support Ticket”**
- Authentication: OAuth2 (configure Zendesk OAuth2 credentials: subdomain, client id/secret, redirect URL as required by n8n)
- Operation: **Create Ticket**
- Use JSON parameters to map:
  - Subject, comment body, requester name/email, tags, priority/status, external_id (thread id)
- Connect from your final gate node to Create Ticket.

11) **Credentials setup**
- **Gmail OAuth2:** connect a Google account with Gmail API enabled and permissions to read inbox/labels.
- **Zendesk OAuth2:** configure Zendesk OAuth app + n8n credentials; ensure permissions allow reading tickets and creating tickets.

12) **Activate workflow**
- Test with a sample email.
- Validate:
  - Spam gets dropped
  - Tags appear correctly in Zendesk
  - Duplicate detection behaves as intended

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works … converts support emails into structured Zendesk tickets … filters spam … extracts details … tags … checks existing tickets … avoids duplicates … creates ticket.” | Sticky note “How It Works” (overview of intended behavior). |
| Setup outline: add trigger → spam detection → set/normalize → tagging → fetch Zendesk tickets → merge + duplicate check → create ticket. | Sticky note “Workflow Setup Steps” (high-level build sequence). |