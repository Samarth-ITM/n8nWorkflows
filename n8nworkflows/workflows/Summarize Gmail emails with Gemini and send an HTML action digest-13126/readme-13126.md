Summarize Gmail emails with Gemini and send an HTML action digest

https://n8nworkflows.xyz/workflows/summarize-gmail-emails-with-gemini-and-send-an-html-action-digest-13126


# Summarize Gmail emails with Gemini and send an HTML action digest

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Summarize Gmail emails with Gemini and send an HTML action digest

**Purpose:**  
Every 12 hours (or on manual run), the workflow fetches recent Gmail emails, keeps only Inbox messages, uses Google Gemini to extract a structured summary/action/priority/recipient, builds a Gmail deep link to each message, aggregates all items, and sends a single HTML digest email. If no emails are found, it sends a “No new emails” HTML message instead.

### 1.1 Triggers
Two entry points: scheduled execution every 12 hours, and manual trigger for testing.

### 1.2 Config & Time Window
Defines the digest recipient and computes a `receivedAfter` timestamp (now minus 12 hours).

### 1.3 Fetch & Prepare Emails
Fetches Gmail messages since `receivedAfter`, normalizes key fields, strips HTML to text, and filters to Inbox messages only.

### 1.4 Empty Run Handling
Checks if any message exists. If none, sends a “no new emails” email.

### 1.5 AI Extraction (Gemini)
For each email, Gemini returns structured JSON via a schema-based parser (summary, action, priority, sender email, recipient).

### 1.6 Build Digest & Send
Builds a Gmail “Open message” deep link using `rfc822msgid`, merges it with AI output, aggregates all items, and sends one HTML digest email.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers
**Overview:** Starts the workflow either on a 12-hour schedule or manually for testing; both converge into the same configuration step.  
**Nodes involved:** `Run every 12 hours`, `When clicking ‘Test workflow’`

#### Node: Run every 12 hours
- **Type / role:** Schedule Trigger — periodic entry point
- **Config:** Runs every **12 hours** (`hoursInterval: 12`)
- **Outputs:** → `Config (edit me)`
- **Version notes:** Schedule Trigger v1.2
- **Failure modes / edge cases:** n8n instance downtime may skip runs unless configured with execution recovery.

#### Node: When clicking ‘Test workflow’
- **Type / role:** Manual Trigger — development/testing entry point
- **Config:** No parameters
- **Outputs:** → `Config (edit me)`
- **Version notes:** Manual Trigger v1
- **Failure modes:** None (only runs when invoked).

---

### Block 2.2 — Config & Time Window
**Overview:** Defines the recipient of the digest and computes a Unix timestamp (ms) for Gmail filtering.  
**Nodes involved:** `Config (edit me)`, `Compute receivedAfter (now - 12h)`

#### Node: Config (edit me)
- **Type / role:** Set — centralized config values
- **Config choices:** Sets `recipientEmail` (currently empty in JSON; must be filled)
- **Key variables:** `recipientEmail`
- **Outputs:** → `Compute receivedAfter (now - 12h)`
- **Version notes:** Set v3.4
- **Failure modes / edge cases:**
  - If `recipientEmail` is blank, both “Send HTML digest email” and “Send no new emails” will fail or send nowhere.

#### Node: Compute receivedAfter (now - 12h)
- **Type / role:** Date & Time — computes time window boundary
- **Config choices:**
  - Date expression: `{{ $now.minus({ hours: 12 }).toISO() }}`
  - Operation: `formatDate`
  - Output format: `x` (Unix timestamp in **milliseconds**)
- **Output field:** `formattedDate` (used by Gmail node)
- **Outputs:** → `Fetch Gmail emails (receivedAfter)`
- **Version notes:** DateTime v2
- **Failure modes / edge cases:**
  - If `$now` is unavailable (rare) or expression errors, Gmail fetch will fail.
  - Using milliseconds is compatible with Gmail node’s `receivedAfter` filter in n8n; if a future node expects seconds, adjust accordingly.

---

### Block 2.3 — Fetch & Prepare Emails
**Overview:** Pulls emails received after the computed time, then normalizes fields and filters down to Inbox messages only.  
**Nodes involved:** `Fetch Gmail emails (receivedAfter)`, `Normalize email fields (From/To/snippet/messageId)`, `Filter: Inbox only`

#### Node: Fetch Gmail emails (receivedAfter)
- **Type / role:** Gmail — read emails
- **Config choices:**
  - Operation: `getAll`
  - Return all: enabled
  - Filter: `receivedAfter = {{ $json.formattedDate }}`
  - `alwaysOutputData: true` (ensures output even if no items)
- **Inputs:** from `Compute receivedAfter (now - 12h)`
- **Outputs:** → `Normalize email fields (From/To/snippet/messageId)`
- **Credentials:** Gmail OAuth2 (must be configured)
- **Version notes:** Gmail v2.1
- **Failure modes / edge cases:**
  - OAuth token expired/invalid.
  - Gmail API quota/rate limits when inbox is large.
  - Message payload fields may differ if “simple” vs non-simple; this node is `simple:false`, so it returns richer fields.

#### Node: Normalize email fields (From/To/snippet/messageId)
- **Type / role:** Set — maps Gmail fields into consistent names used downstream
- **Config choices / outputs created:**
  - `id = {{$json.id}}`
  - `Body = {{$json.text}}`
  - `From = {{$json.from.text}}`
  - `TO = {{$json.to.text}}`
  - `Subject = {{$json.subject}}`
  - `messageId = {{$json.messageId}}`
  - `location = {{$json.labelIds.includes('INBOX') ? 'Inbox' : ($json.labelIds.includes('SENT') ? 'Sent' : 'Other') }}`
  - `html = {{$json.html.replace(/<[^>]*>/g, '')}}` (strips HTML tags; despite the name, output is plain text)
- **Inputs:** from `Fetch Gmail emails (receivedAfter)`
- **Outputs:** → `Filter: Inbox only`
- **Version notes:** Set v3.4
- **Failure modes / edge cases:**
  - If `$json.html` is null/undefined, `.replace(...)` throws. Consider guarding: `($json.html ?? '').replace(...)`.
  - If `labelIds` is missing, `.includes` throws; guard similarly.
  - Field naming inconsistency: downstream prompt says “Snippet” but uses this node’s `html` field.

#### Node: Filter: Inbox only
- **Type / role:** IF — keeps only Inbox items
- **Condition:** `{{$json.location}} equals "Inbox"`
- **True output:** → `If: any emails found?`
- **False output:** → `No Operation, do nothing1`
- **Version notes:** IF v2.2
- **Failure modes / edge cases:**
  - If `location` missing, condition fails (routes to false), possibly dropping valid items.

---

### Block 2.4 — Empty Run Handling
**Overview:** Determines whether there are any emails to process; if none, sends a dedicated “no new emails” HTML email.  
**Nodes involved:** `If: any emails found?`, `Send “no new emails” email`, `No Operation, do nothing`

#### Node: If: any emails found?
- **Type / role:** IF — checks existence of an email `id`
- **Condition:** `exists( {{$json.id}} )`
- **True output:** → `Build Gmail “Open message” link` and → `AI: extract summary, action, priority` (parallel per item)
- **False output:** → `Send “no new emails” email`
- **Version notes:** IF v2.2
- **Failure modes / edge cases:**
  - If Gmail returns `alwaysOutputData` with an empty item that lacks `id`, it will correctly route to “no emails”.
  - If multiple items: IF runs per item; typically, empty runs yield one item without `id`.

#### Node: Send “no new emails” email
- **Type / role:** Gmail — send an email
- **Config choices:**
  - To: `{{ $('Config (edit me)').item.json.recipientEmail }}`
  - Subject: `Email Summary`
  - HTML body: “No new emails in the past 12 hours.” template
- **Outputs:** → `No Operation, do nothing`
- **Credentials:** Gmail OAuth2
- **Version notes:** Gmail v2.1
- **Failure modes / edge cases:**
  - Missing/blank `recipientEmail`
  - OAuth/permission errors
  - Gmail may rewrite HTML; keep inline styles for email client compatibility.

#### Node: No Operation, do nothing
- **Type / role:** NoOp — terminator / visual end node
- **Config:** None
- **Inputs:** from `Send “no new emails” email`
- **Outputs:** None (end)
- **Version notes:** NoOp v1

---

### Block 2.5 — AI Extraction (Gemini + Structured Output)
**Overview:** For each email, an AI Agent (LangChain-based) uses Gemini and a structured schema parser to output consistent JSON fields used in the digest.  
**Nodes involved:** `Google Gemini Chat Model`, `Structured Output Parser`, `AI: extract summary, action, priority`

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain LLM connector — provides the Gemini chat model
- **Config:** Default options (model selection is handled via credentials/account availability)
- **Connections:** Provides `ai_languageModel` input to `AI: extract summary, action, priority`
- **Version notes:** Gemini chat model node v1
- **Failure modes / edge cases:**
  - Missing Gemini credentials
  - Model unavailable in region/account
  - Rate limits/timeouts

#### Node: Structured Output Parser
- **Type / role:** LangChain output parser — enforces schema
- **Config:** Schema example defines an **array** with objects containing:
  - `email`, `action`, `priority (high|normal|low)`, `summary`, `recipient`
- **Connections:** Provides `ai_outputParser` to `AI: extract summary, action, priority`
- **Version notes:** Output parser v1.3
- **Failure modes / edge cases:**
  - If the model returns non-conforming JSON, parsing fails and triggers agent retry (since agent has retry enabled).
  - Schema says array-of-objects; later HTML expects `output[0]`.

#### Node: AI: extract summary, action, priority
- **Type / role:** LangChain Agent — prompts model and calls parser tool
- **Config choices:**
  - PromptType: “define” (custom prompt)
  - `hasOutputParser: true`
  - `retryOnFail: true`, wait between tries: 500ms
  - Prompt uses data from `If: any emails found?` node:
    - From: `{{ $('If: any emails found?').item.json.From }}`
    - To: `{{ $('If: any emails found?').item.json.TO }}`
    - Snippet: `{{ $('If: any emails found?').item.json.html }}`
  - Extraction rules:
    - `sender_email` rule described, but the parser schema field is named `email` (so the agent must map sender email into `email`)
    - Priority must be one of `high|normal|low`
    - Recipient must be just email address
- **Inputs:** from `If: any emails found?` (main), plus model + parser via AI connections
- **Outputs:** → `Merge: link + AI result` (to input index 1)
- **Version notes:** Agent v2.2
- **Failure modes / edge cases:**
  - Prompt inconsistency: mentions **Snippet as html**; actually `html` field is stripped text.
  - If `From` or `TO` formats don’t contain an email, extraction can be wrong; consider regex fallback.
  - If `message body` is empty (no `html`), the model may output vague summaries.

---

### Block 2.6 — Build Gmail Link, Merge, Aggregate, Send Digest
**Overview:** Builds a Gmail deep link per email, merges it with AI output, aggregates all items into one list, then sends a styled HTML digest.  
**Nodes involved:** `Build Gmail “Open message” link`, `No Operation, do nothing2`, `Merge: link + AI result`, `Aggregate: build digest list`, `Send HTML digest email`

#### Node: Build Gmail “Open message” link
- **Type / role:** Set — constructs per-email deep link into Gmail UI
- **Key expression:**
  - `link = https://mail.google.com/mail/u/{{ $json.TO.match(emailRegex)[0] }}/#search/rfc822msgid:{{ encodeURIComponent($json.messageId.slice(1, -1)) }}`
  - Extracts first email address from `TO`
  - Uses `messageId` with angle brackets removed via `slice(1, -1)`
  - URL-encodes messageId for safety
- **Inputs:** from `If: any emails found?` (true branch)
- **Outputs:** → `No Operation, do nothing2`
- **Version notes:** Set v3.4
- **Failure modes / edge cases:**
  - If `TO.match(...)` returns null (no email found), indexing `[0]` throws.
  - If `messageId` not wrapped in `< >`, `slice(1,-1)` corrupts it; safer: trim only if it starts/ends with brackets.

#### Node: No Operation, do nothing2
- **Type / role:** NoOp — used to line up merge inputs (acts as pass-through)
- **Inputs:** from `Build Gmail “Open message” link`
- **Outputs:** → `Merge: link + AI result` (to input index 0)
- **Version notes:** NoOp v1

#### Node: Merge: link + AI result
- **Type / role:** Merge — combines two item streams by position
- **Config choices:**
  - Mode: `combine`
  - Combine by: `position` (item 0 with item 0, etc.)
- **Inputs:**
  - Input 0: link stream via `No Operation, do nothing2`
  - Input 1: AI output stream from `AI: extract summary, action, priority`
- **Outputs:** → `Aggregate: build digest list`
- **Version notes:** Merge v3.2
- **Failure modes / edge cases:**
  - If AI node fails for one item but link node succeeds (or vice versa), stream positions can misalign or reduce output.
  - Ensure both branches process the same number of items in the same order.

#### Node: Aggregate: build digest list
- **Type / role:** Aggregate — collects all items into one object for email rendering
- **Config:** `aggregateAllItemData` (puts all items under a single array, typically `data`)
- **Inputs:** from `Merge: link + AI result`
- **Outputs:** → `Send HTML digest email`
- **Version notes:** Aggregate v1
- **Failure modes / edge cases:**
  - Very large digests can produce oversized emails; consider limiting email count.

#### Node: Send HTML digest email
- **Type / role:** Gmail — sends the final digest
- **Config choices:**
  - To: `{{ $('Config (edit me)').item.json.recipientEmail }}`
  - Subject: `Email Summary`
  - HTML body:
    - Iterates `{{ $json.data.map(item => { ... }).join('') }}`
    - Reads AI result at `item.output[0]` (expects parser output array)
    - Priority color-coding:
      - high: red
      - normal: green
      - low: gray
    - Uses `item.link` for “Open message”
  - Options: attribution disabled
- **Inputs:** from `Aggregate: build digest list`
- **Credentials:** Gmail OAuth2
- **Version notes:** Gmail v2.1
- **Failure modes / edge cases:**
  - If aggregate output key is not `data` (depends on Aggregate node behavior/version), the template breaks.
  - If AI output shape differs (not `output[0]`), sender/summary/action fields will render empty.
  - Gmail may sanitize some HTML/CSS; mostly safe due to table-based layout.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow commentary |  |  | ## Workflow overview / Who it’s for / What this workflow does / Recommended AI model / Required setup before running / Requirements / Customizing this workflow (includes “[web:170]”) |
| Sticky Note3 | Sticky Note | Workflow commentary |  |  | ### Build digest & send: Creates Gmail deep link per email, aggregates results, and sends one HTML digest. |
| Sticky Note4 | Sticky Note | Workflow commentary |  |  | ### Fetch & prepare emails Fetch Gmail emails since receivedAfter, then normalize fields (From/To/snippet/messageId) and keep Inbox only. |
| Sticky Note5 | Sticky Note | Workflow commentary |  |  | ### Triggers Runs every 12 hours (schedule) or via manual test. Both paths go into the same flow. |
| Sticky Note6 | Sticky Note | Workflow commentary |  |  | ### AI extraction: Gemini analyzes each email and outputs structured JSON: summary, action, priority, recipient. |
| Sticky Note7 | Sticky Note | Workflow commentary |  |  | ### Config & Time window **Config (edit me)** Set `recipientEmail` here. All Gmail nodes use it. **Time window** "Now - 12h" timestamp… |
| Sticky Note8 | Sticky Note | Workflow commentary |  |  | ### Empty run handling / Send "no new emails" / Build Gmail link (deep link via rfc822msgid) |
| When clicking ‘Test workflow’ | Manual Trigger | Manual entry point |  | Config (edit me) | ### Triggers Runs every 12 hours (schedule) or via manual test. Both paths go into the same flow. |
| Run every 12 hours | Schedule Trigger | Scheduled entry point |  | Config (edit me) | ### Triggers Runs every 12 hours (schedule) or via manual test. Both paths go into the same flow. |
| Config (edit me) | Set | Stores recipient configuration | Run every 12 hours; When clicking ‘Test workflow’ | Compute receivedAfter (now - 12h) | ### Config & Time window **Config (edit me)** Set `recipientEmail` here… |
| Compute receivedAfter (now - 12h) | Date & Time | Compute lookback timestamp | Config (edit me) | Fetch Gmail emails (receivedAfter) | ### Config & Time window … "Now - 12h" timestamp… |
| Fetch Gmail emails (receivedAfter) | Gmail | Fetch emails after timestamp | Compute receivedAfter (now - 12h) | Normalize email fields (From/To/snippet/messageId) | ### Fetch & prepare emails Fetch Gmail emails since receivedAfter… |
| Normalize email fields (From/To/snippet/messageId) | Set | Normalize and derive fields | Fetch Gmail emails (receivedAfter) | Filter: Inbox only | ### Fetch & prepare emails Fetch Gmail emails since receivedAfter… |
| Filter: Inbox only | IF | Keep only Inbox items | Normalize email fields (From/To/snippet/messageId) | If: any emails found?; No Operation, do nothing1 | ### Fetch & prepare emails …keep Inbox only. |
| No Operation, do nothing1 | NoOp | Sink for non-Inbox items | Filter: Inbox only (false) |  | ### Fetch & prepare emails …keep Inbox only. |
| If: any emails found? | IF | Branch: process vs “no emails” | Filter: Inbox only (true) | Build Gmail “Open message” link; AI: extract summary, action, priority; Send “no new emails” email | ### Empty run handling … True → process. False → send "No new emails" HTML. |
| Send “no new emails” email | Gmail | Send empty-run notification | If: any emails found? (false) | No Operation, do nothing | ### Empty run handling / Send "no new emails" email Uses recipientEmail from Config… |
| No Operation, do nothing | NoOp | Terminator | Send “no new emails” email |  | ### Empty run handling / Send "no new emails" email… |
| Build Gmail “Open message” link | Set | Create Gmail deep link | If: any emails found? (true) | No Operation, do nothing2 | ### Build Gmail link Build Gmail "Open message" link… |
| No Operation, do nothing2 | NoOp | Pass-through to align merge | Build Gmail “Open message” link | Merge: link + AI result | ### Build digest & send: Creates Gmail deep link per email… |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM provider for agent |  | AI: extract summary, action, priority (AI connection) | ### AI extraction: Gemini analyzes each email… |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce schema output |  | AI: extract summary, action, priority (AI connection) | ### AI extraction: Gemini analyzes each email… |
| AI: extract summary, action, priority | LangChain Agent | AI extraction per email | If: any emails found? (true) | Merge: link + AI result | ### AI extraction: Gemini analyzes each email… |
| Merge: link + AI result | Merge | Combine link + AI output | No Operation, do nothing2; AI: extract summary, action, priority | Aggregate: build digest list | ### Build digest & send: Creates Gmail deep link per email… |
| Aggregate: build digest list | Aggregate | Collect all items into one list | Merge: link + AI result | Send HTML digest email | ### Build digest & send: Creates Gmail deep link per email… |
| Send HTML digest email | Gmail | Send final HTML digest | Aggregate: build digest list |  | ### Build digest & send: Creates Gmail deep link per email… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger nodes**
1. Add **Schedule Trigger** named **“Run every 12 hours”**  
   - Set interval: **Every 12 hours**.
2. Add **Manual Trigger** named **“When clicking ‘Test workflow’”**.

2) **Create config node**
3. Add a **Set** node named **“Config (edit me)”**  
   - Add field `recipientEmail` (string) and set it to your target address (e.g., `you@domain.com`).
4. Connect both triggers → **Config (edit me)**.

3) **Compute time window**
5. Add **Date & Time** node named **“Compute receivedAfter (now - 12h)”**  
   - Operation: **Format Date**
   - Date: `{{ $now.minus({ hours: 12 }).toISO() }}`
   - Format: `x`
6. Connect **Config (edit me)** → **Compute receivedAfter (now - 12h)**.

4) **Fetch emails from Gmail**
7. Add **Gmail** node named **“Fetch Gmail emails (receivedAfter)”**  
   - Operation: **Get Many / GetAll**
   - Return All: **true**
   - Filters → **Received After**: `{{ $json.formattedDate }}`
   - Set **Always Output Data** to true (so the flow can handle empty runs).
   - **Credentials:** Create/select **Gmail OAuth2** credentials with Gmail API access.
8. Connect **Compute receivedAfter (now - 12h)** → **Fetch Gmail emails (receivedAfter)**.

5) **Normalize fields**
9. Add **Set** node named **“Normalize email fields (From/To/snippet/messageId)”** and map:
   - `id` = `{{ $json.id }}`
   - `Body` = `{{ $json.text }}`
   - `From` = `{{ $json.from.text }}`
   - `TO` = `{{ $json.to.text }}`
   - `Subject` = `{{ $json.subject }}`
   - `messageId` = `{{ $json.messageId }}`
   - `location` = `{{ $json.labelIds.includes('INBOX') ? 'Inbox' : ($json.labelIds.includes('SENT') ? 'Sent' : 'Other') }}`
   - `html` = `{{ $json.html.replace(/<[^>]*>/g, '') }}`
10. Connect **Fetch Gmail emails (receivedAfter)** → **Normalize…**.

6) **Filter Inbox only**
11. Add **IF** node named **“Filter: Inbox only”**  
   - Condition: `{{ $json.location }}` **equals** `Inbox`
12. Connect **Normalize…** → **Filter: Inbox only**.
13. Add **NoOp** named **“No Operation, do nothing1”** and connect it to the **false** output (optional sink).

7) **Check empty run**
14. Add **IF** node named **“If: any emails found?”**  
   - Condition: `exists({{ $json.id }})`
15. Connect **Filter: Inbox only (true)** → **If: any emails found?**.

8) **No-email path**
16. Add **Gmail** node named **“Send “no new emails” email”**  
   - To: `{{ $('Config (edit me)').item.json.recipientEmail }}`
   - Subject: `Email Summary`
   - Message: paste your HTML “No new emails…” template
   - Credentials: same Gmail OAuth2
17. Connect **If: any emails found? (false)** → **Send “no new emails” email**.
18. Add **NoOp** named **“No Operation, do nothing”** and connect after the send node (optional end).

9) **Build Gmail deep link (per email)**
19. Add **Set** node named **“Build Gmail “Open message” link”**  
   - Add field `link` with:
     ```
     https://mail.google.com/mail/u/{{ $json.TO.match(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g)[0] }}/#search/rfc822msgid:{{ encodeURIComponent($json.messageId.slice(1, -1)) }}
     ```
20. Connect **If: any emails found? (true)** → **Build Gmail “Open message” link**.

10) **AI extraction setup (Gemini + schema)**
21. Add **Google Gemini Chat Model** node named **“Google Gemini Chat Model”**  
   - Configure **Gemini API credentials** (Google AI Studio / Vertex as applicable in your n8n).
   - Select an available Gemini model (the note recommends “Gemini 2.5 Flash” if available).
22. Add **Structured Output Parser** node named **“Structured Output Parser”**  
   - Provide schema example (array of objects) with fields: `email`, `action`, `priority`, `summary`, `recipient`.
23. Add **AI Agent** node named **“AI: extract summary, action, priority”**  
   - Prompt: use the workflow’s provided instruction text (ensure it calls the Structured Output Parser tool).
   - Enable structured output / output parser.
   - Enable retry on fail (optional but recommended).
24. Connect:
   - **Google Gemini Chat Model** → Agent via **AI Language Model** connection.
   - **Structured Output Parser** → Agent via **AI Output Parser** connection.
25. Connect **If: any emails found? (true)** → **AI: extract…** (main connection).

11) **Merge link + AI output**
26. Add **NoOp** node named **“No Operation, do nothing2”** (pass-through).
27. Connect **Build Gmail “Open message” link** → **No Operation, do nothing2**.
28. Add **Merge** node named **“Merge: link + AI result”**
   - Mode: **Combine**
   - Combine by: **Position**
29. Connect:
   - **No Operation, do nothing2** → Merge input **0**
   - **AI: extract…** → Merge input **1**

12) **Aggregate and send digest**
30. Add **Aggregate** node named **“Aggregate: build digest list”**
   - Mode: **Aggregate All Item Data**
31. Connect **Merge** → **Aggregate**.
32. Add **Gmail** node named **“Send HTML digest email”**
   - To: `{{ $('Config (edit me)').item.json.recipientEmail }}`
   - Subject: `Email Summary`
   - Message: paste the provided HTML template that maps over `{{ $json.data.map(...) }}`
   - Credentials: Gmail OAuth2
33. Connect **Aggregate** → **Send HTML digest email**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Recommended AI model: “Gemini 2.5 Flash” (or closest available). | Mentioned in workflow overview sticky note (contains placeholder “[web:170]” without a URL). |
| Before running: configure Gmail OAuth2 credentials for Fetch/Send nodes; configure Gemini credentials; set recipient email in Config and/or Send nodes. | Workflow overview sticky note. |
| Customization ideas: change lookback window (12h→24h), adjust AI rules, modify HTML design, remove Inbox-only filter to include other labels. | Workflow overview sticky note. |