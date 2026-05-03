Send AI-generated Gmail auto replies with GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/send-ai-generated-gmail-auto-replies-with-gpt-4o-mini-and-google-sheets-12652


# Send AI-generated Gmail auto replies with GPT-4o-mini and Google Sheets

## 1. Workflow Overview

**Workflow name:** *Gmail AI Auto Reply with Duplicate Prevention*  
**Stated title:** *Send AI-generated Gmail auto replies with GPT-4o-mini and Google Sheets*  

**Purpose:**  
Automatically generate and send **AI-written English replies** to incoming Gmail messages while **preventing duplicate replies** by checking (1) a **Google Sheets reply history** and (2) **recent sent mail** in Gmail.

**Target use cases:**
- Auto-responding to inbound emails that are not newsletters/promotions
- Avoiding replying multiple times to the same person within a configured time window
- Producing concise, human-sounding replies with GPT-4o-mini

### 1.1 Email Intake & Configuration
Receives new emails via Gmail Trigger, then loads configuration values (ignore rules, thresholds, sheet name, etc.) and normalizes the email fields for consistent downstream processing.

### 1.2 Ignore Filtering (Stop early)
Filters out promotional/newsletter-like emails based on sender, subject keywords, and Gmail category labels.

### 1.3 Duplicate Prevention – Google Sheets
Extracts the sender email address and checks if that address already exists in a Google Sheet (reply history). If found, the workflow stops.

### 1.4 Duplicate Prevention – Gmail Sent Mail Search
If not found in Sheets, searches Gmail “Sent” to see whether any email was recently sent to the same sender within a configured number of days. If found, the workflow stops.

### 1.5 AI Reply Generation & Reply Send
If no duplicate is detected, sends the inbound email to GPT-4o-mini with constraints (concise, no guessing, <120 words), then replies to the original message in Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake & Normalization
**Overview:**  
Captures new incoming emails and standardizes key fields (from/subject/body/labels) into predictable lowercase/string forms for filtering and downstream checks.

**Nodes involved:**
- **Gmail Trigger - New Emails**
- **Config - Rules & Thresholds**
- **Normalize - Email Payload**

#### Node: Gmail Trigger - New Emails
- **Type / role:** Gmail Trigger — entry point; polls Gmail for new emails.
- **Key configuration (interpreted):**
  - Polling schedule: **every minute**
  - No specific filters configured in the trigger (filters object is empty).
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Connections:**
  - Output → **Config - Rules & Thresholds**
- **Failure/edge cases:**
  - OAuth token expiration / revoked consent
  - Gmail API quota limits
  - Trigger may pull emails you didn’t intend if no filters/labels are set

#### Node: Config - Rules & Thresholds
- **Type / role:** Set — stores “global” constants used later via `$json`.
- **Key configuration choices:**
  - `ignoreListIncludes`: keywords for senders to ignore (unsubscribe/newsletter/no-reply/etc.)
  - `ignoreSubjectIncludes`: keywords for subjects to ignore (sale/discount/newsletter/digest/etc.)
  - `ignoreLabelsIncludes`: Gmail categories to ignore (`CATEGORY_PROMOTIONS|CATEGORY_SOCIAL`)
  - `importantReplyHistoryDays`: **365** days lookback for sent-mail duplicate check
  - `aiRunIfUncertain`: **true** (declared but not used elsewhere in this workflow)
  - `urgentScoreThreshold`, `importantScoreThreshold`: declared but not used elsewhere
  - `alertEmailTo`, `alertEmailSubjectPrefix`: declared but not used elsewhere
  - `sheetName`: `inbox_audit_log` (declared but the Sheets node is currently pointed to “シート1”; see edge case below)
- **Connections:**
  - Input ← Gmail Trigger - New Emails
  - Output → Normalize - Email Payload
- **Failure/edge cases:**
  - Confusion if later nodes don’t actually reference these config fields (some are unused)
  - `sheetName` is not used by the Google Sheets node in this JSON (possible drift between config and implementation)

#### Node: Normalize - Email Payload
- **Type / role:** Set — normalizes incoming email fields and creates helper fields.
- **Key fields created (expressions):**
  - `from_raw` = `{{$json.from}}`
  - `from_lower` = `{{($json.from || '').toLowerCase()}}`
  - `subject_raw` / `subject_lower`
  - `body_raw` = `{{$json.textPlain || $json.snippet || ''}}`
  - `body_lower` = lowercase variant
  - `labels` = `{{($json.labelIds || []).join('|')}}`
  - Includes other fields: **true** (keeps original trigger payload)
- **Connections:**
  - Input ← Config - Rules & Thresholds
  - Output → IF - Ignore Filter
- **Failure/edge cases:**
  - Some triggers may not provide `textPlain`; fallback to `snippet` is used (good), but snippet can be incomplete
  - If `labelIds` missing, defaults to empty array (safe)

---

### Block 2 — Ignore Unwanted Emails
**Overview:**  
Stops processing immediately for emails matching known non-actionable patterns (newsletters/promotions/social updates).

**Nodes involved:**
- **IF - Ignore Filter**

#### Node: IF - Ignore Filter
- **Type / role:** IF — routes emails based on ignore matching.
- **Logic (OR combinator):** If *any* condition matches, it is considered “ignored”.
  1. `from_lower` matches a regex containing: unsubscribe/newsletter/no-reply/noreply/notification/do-not-reply/marketing/promo/promotion  
  2. `subject_lower` matches sale/discount/coupon/points/promotion/campaign/special offer/deal/new arrivals/update/digest/weekly/daily/newsletter  
  3. `labels` contains promotions or social: `CATEGORY_PROMOTIONS|CATEGORY_SOCIAL`
- **Connections:**
  - Input ← Normalize - Email Payload
  - **True output (ignored):** not connected (workflow stops)
  - **False output (not ignored):** → Code - Extract Sender Email
- **Failure/edge cases:**
  - Regex false positives/negatives depending on sender formats and languages
  - Gmail labels vary; if categories aren’t present in `labelIds`, label filtering may be ineffective
  - Because “true path” is unconnected, ignored emails simply end silently (expected)

---

### Block 3 — Reply History Check (Google Sheets)
**Overview:**  
Extracts a canonical sender email and checks a Google Sheet for prior replies; if found, stops to avoid duplicates.

**Nodes involved:**
- **Code - Extract Sender Email**
- **Get row(s) in sheet**
- **IF - Found in Reply History?**

#### Node: Code - Extract Sender Email
- **Type / role:** Code — parses the sender email address from the `From:` header.
- **Key logic:**
  - Extracts `<email@domain>` if present; else uses raw string
  - Trims and lowercases result into `sender_email`
- **Connections:**
  - Input ← IF - Ignore Filter (false path)
  - Output → Get row(s) in sheet
- **Failure/edge cases:**
  - Non-standard `From` formats (multiple addresses, encoded names) may parse incorrectly
  - If `from_raw` is empty, `sender_email` becomes empty string; downstream lookups may match unintended rows if the sheet contains blanks

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets — lookup rows by sender email.
- **Key configuration choices:**
  - Operation: “Get row(s)” (read)
  - Filter: lookup where `sender_email` column equals `{{$json.sender_email}}`
  - Document: spreadsheet named “reply_history” (ID shown in node)
  - Sheet tab: currently set to **“シート1”** (gid=0)
- **Credentials:** Google Sheets OAuth2
- **Connections:**
  - Input ← Code - Extract Sender Email
  - Output → IF - Found in Reply History?
- **Failure/edge cases:**
  - If the sheet does not have a `sender_email` header column, lookup fails or returns nothing
  - If multiple matching rows exist, the node returns multiple items; downstream IF uses `$input.all().length`
  - Note mismatch: Config node sets `sheetName = inbox_audit_log` but this node uses “シート1” (implementation drift)

#### Node: IF - Found in Reply History?
- **Type / role:** IF — detects whether any matching sheet rows were returned.
- **Condition:** `{{$input.all().length > 0}}` is true
- **Connections:**
  - Input ← Get row(s) in sheet
  - **True output (found in sheet):** not connected (workflow stops)
  - **False output (not found):** → Gmail - Search Sent to Sender
- **Failure/edge cases:**
  - If the Google Sheets node errors, execution stops unless error handling is configured (not shown here)
  - If Sheets returns items but not in expected format, length check still works (robust)

---

### Block 4 — Recent Sent Email Check (Gmail)
**Overview:**  
Searches the Gmail Sent folder for emails sent to the sender in the last N days (default 365). If any exist, stops.

**Nodes involved:**
- **Gmail - Search Sent to Sender**
- **IF - Sent history exists?**

#### Node: Gmail - Search Sent to Sender
- **Type / role:** Gmail — “getAll” operation; searches messages with a Gmail query.
- **Key configuration choices:**
  - Limit: **20**
  - Query `q` (expression):  
    `in:sent to:<sender_email> newer_than:<importantReplyHistoryDays>d`
  - Uses `importantReplyHistoryDays` from earlier config carried in `$json` (365 by default).
- **Credentials:** Gmail OAuth2
- **Connections:**
  - Input ← IF - Found in Reply History? (false path)
  - Output → IF - Sent history exists?
- **Failure/edge cases:**
  - Gmail search syntax: `to:` query should work, but edge cases with aliases/plus-addressing may reduce matches
  - If `sender_email` is empty, query becomes `in:sent to: newer_than:365d` which can return unintended results
  - Gmail API quota/auth failures

#### Node: IF - Sent history exists?
- **Type / role:** IF — decides whether to proceed to AI generation based on search results.
- **Condition:** `{{$items().length > 0}}` is true
- **Connections:**
  - Input ← Gmail - Search Sent to Sender
  - **True output (sent history exists):** not connected (workflow stops)
  - **False output (no sent history):** → Message a model
- **Failure/edge cases:**
  - `$items().length` depends on the node returning items; if Gmail node returns 0 items, it proceeds as intended

---

### Block 5 — AI Reply Generation & Sending
**Overview:**  
Generates a constrained English reply using GPT-4o-mini and sends it as a Gmail reply to the original message.

**Nodes involved:**
- **Message a model**
- **Reply to a message**

#### Node: Message a model
- **Type / role:** OpenAI (LangChain) — prompts GPT-4o-mini to draft a reply.
- **Model:** `gpt-4o-mini`
- **Prompt content (key constraints):**
  - Polite, natural English email reply
  - Do not guess missing information
  - Clear and concise, under 120 words
  - End with one clear next action
  - Includes original `{{$json.subject}}` and `{{$json.text || $json.snippet}}`
- **Credentials:** OpenAI API (“Kota1”)
- **Connections:**
  - Input ← IF - Sent history exists? (false path)
  - Output → Reply to a message
- **Failure/edge cases:**
  - Missing `text` field: uses `snippet` fallback
  - Token limits or OpenAI API errors/timeouts
  - Output mapping expectation: downstream node uses `{{$json.text}}` as the reply body; this relies on this node outputting the generated text in a `text` property

#### Node: Reply to a message
- **Type / role:** Gmail — replies to the original inbound message.
- **Key configuration choices:**
  - Operation: **reply**
  - `messageId`: `{{$json.id}}` (expects original inbound message id to still be present)
  - `message`: `{{$json.text}}` (expects AI output text field)
  - Email type: text
- **Credentials:** Gmail OAuth2
- **Connections:**
  - Input ← Message a model
  - Output: none
- **Failure/edge cases:**
  - If the OpenAI node overwrites or does not carry forward the original `id`, the reply will fail (missing/invalid messageId)
  - If AI output isn’t stored in `$json.text`, the sent message will be empty or expression will error
  - Gmail API may reject replies to certain system messages or restricted threads

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger - New Emails | n8n-nodes-base.gmailTrigger | Entry trigger: poll Gmail for new emails | — | Config - Rules & Thresholds | ## Step 1 – Email Intake & Normalization; Receive new Gmail emails, load global rules and thresholds, and normalize email data into a consistent structure for stable downstream processing. |
| Config - Rules & Thresholds | n8n-nodes-base.set | Store workflow rules/thresholds/constants | Gmail Trigger - New Emails | Normalize - Email Payload | ## Step 1 – Email Intake & Normalization; Receive new Gmail emails, load global rules and thresholds, and normalize email data into a consistent structure for stable downstream processing. |
| Normalize - Email Payload | n8n-nodes-base.set | Normalize fields (from/subject/body/labels) | Config - Rules & Thresholds | IF - Ignore Filter | ## Step 1 – Email Intake & Normalization; Receive new Gmail emails, load global rules and thresholds, and normalize email data into a consistent structure for stable downstream processing. |
| IF - Ignore Filter | n8n-nodes-base.if | Filter out newsletters/promos/social | Normalize - Email Payload | (false) Code - Extract Sender Email | ## Step 2 – Ignore Unwanted Emails; Filter out newsletters, promotions, and irrelevant emails. Messages that match ignore rules are stopped immediately. |
| Code - Extract Sender Email | n8n-nodes-base.code | Parse canonical sender email from From header | IF - Ignore Filter (false) | Get row(s) in sheet | ## Step 3 – Reply History Check (Google Sheet); Extract the sender’s email address and check whether it already exists in the Google Sheet reply history. If found, no reply is sent. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Lookup sender in Sheets reply history | Code - Extract Sender Email | IF - Found in Reply History? | ## Step 3 – Reply History Check (Google Sheet); Extract the sender’s email address and check whether it already exists in the Google Sheet reply history. If found, no reply is sent. |
| IF - Found in Reply History? | n8n-nodes-base.if | Stop if sender exists in reply history sheet | Get row(s) in sheet | (false) Gmail - Search Sent to Sender | ## Step 3 – Reply History Check (Google Sheet); Extract the sender’s email address and check whether it already exists in the Google Sheet reply history. If found, no reply is sent. |
| Gmail - Search Sent to Sender | n8n-nodes-base.gmail | Search Sent mail to prevent duplicates | IF - Found in Reply History? (false) | IF - Sent history exists? | ## Step 4 – Recent Sent Email Check (Gmail); Search Gmail for recently sent messages to the same sender. If a recent reply exists within the defined time window, the workflow stops to prevent duplicate replies. |
| IF - Sent history exists? | n8n-nodes-base.if | Stop if sent-mail history exists | Gmail - Search Sent to Sender | (false) Message a model | ## Step 4 – Recent Sent Email Check (Gmail); Search Gmail for recently sent messages to the same sender. If a recent reply exists within the defined time window, the workflow stops to prevent duplicate replies. |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Generate reply text with GPT-4o-mini | IF - Sent history exists? (false) | Reply to a message | ## Step 5 – AI Reply Generation & Sending; Generate a clear English reply using AI and send it as a reply to the original Gmail message. This step runs only when no duplicate reply is detected. |
| Reply to a message | n8n-nodes-base.gmail | Send Gmail reply to original message | Message a model | — | ## Step 5 – AI Reply Generation & Sending; Generate a clear English reply using AI and send it as a reply to the original Gmail message. This step runs only when no duplicate reply is detected. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Overview – Workflow Purpose; This workflow automatically replies to important incoming Gmail messages. It avoids duplicate replies by checking both a Google Sheet reply history and recent sent emails in Gmail before responding. Only emails that pass all filters receive an AI-generated English reply. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Step 1 – Email Intake & Normalization; Receive new Gmail emails, load global rules and thresholds, and normalize email data into a consistent structure for stable downstream processing. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Step 2 – Ignore Unwanted Emails; Filter out newsletters, promotions, and irrelevant emails. Messages that match ignore rules are stopped immediately. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Step 3 – Reply History Check (Google Sheet); Extract the sender’s email address and check whether it already exists in the Google Sheet reply history. If found, no reply is sent. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Step 4 – Recent Sent Email Check (Gmail); Search Gmail for recently sent messages to the same sender. If a recent reply exists within the defined time window, the workflow stops to prevent duplicate replies. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment / documentation | — | — | ## Step 5 – AI Reply Generation & Sending; Generate a clear English reply using AI and send it as a reply to the original Gmail message. This step runs only when no duplicate reply is detected. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: *Gmail AI Auto Reply with Duplicate Prevention*
   - Set it to inactive while building/testing.

2) **Add Gmail Trigger**
   - Node: **Gmail Trigger**
   - Name: **Gmail Trigger - New Emails**
   - Polling: **Every minute**
   - Credentials: connect **Gmail OAuth2** (Google account to monitor)
   - Connect: Gmail Trigger → Config node (next step)

3) **Add configuration Set node**
   - Node: **Set**
   - Name: **Config - Rules & Thresholds**
   - Add fields:
     - `ignoreListIncludes` (string): `unsubscribe|newsletter|no-reply|noreply|notification|do-not-reply|marketing|promo|promotion`
     - `ignoreSubjectIncludes` (string): `sale|discount|coupon|points|promotion|campaign|special offer|deal|new arrivals|update|digest|weekly|daily|newsletter`
     - `ignoreLabelsIncludes` (string): `CATEGORY_PROMOTIONS|CATEGORY_SOCIAL`
     - `importantReplyHistoryDays` (number): `365`
     - `aiRunIfUncertain` (boolean): `true`
     - `urgentScoreThreshold` (number): `80`
     - `importantScoreThreshold` (number): `60`
     - `alertEmailTo` (string): `user@example.com`
     - `alertEmailSubjectPrefix` (string): `[IMPORTANT INBOX]`
     - `sheetName` (string): `inbox_audit_log` *(note: only useful if you actually reference it later)*
   - Keep **Include Other Fields** enabled (default is usually enabled in Set v3+ if configured).
   - Connect: Config → Normalize node

4) **Add normalization Set node**
   - Node: **Set**
   - Name: **Normalize - Email Payload**
   - Enable: **Include Other Fields = true**
   - Add fields (expressions):
     - `from_raw`: `={{$json.from}}`
     - `from_lower`: `={{($json.from || '').toLowerCase()}}`
     - `subject_raw`: `={{$json.subject}}`
     - `subject_lower`: `={{($json.subject || '').toLowerCase()}}`
     - `body_raw`: `={{$json.textPlain || $json.snippet || ''}}`
     - `body_lower`: `={{($json.textPlain || $json.snippet || '').toLowerCase()}}`
     - `labels`: `={{($json.labelIds || []).join('|')}}`
   - Connect: Normalize → IF Ignore Filter

5) **Add IF node for ignore filtering**
   - Node: **IF**
   - Name: **IF - Ignore Filter**
   - Condition group: **OR**
   - Conditions (String → Regex):
     - Left: `={{$json.from_lower}}`  
       Right: `={{"(?:^|[^a-z0-9])(unsubscribe|newsletter|no-reply|noreply|notification|do-not-reply|marketing|promo|promotion)(?:[^a-z0-9]|$)"}}`
     - Left: `={{$json.subject_lower}}`  
       Right: `={{"(sale|discount|coupon|points|promotion|campaign|special\\s+offer|deal|new\\s+arrivals|update|digest|weekly|daily|newsletter)"}}`
     - Left: `={{$json.labels}}`  
       Right: `={{"(CATEGORY_PROMOTIONS|CATEGORY_SOCIAL)"}}`
   - Connect the **false** output (not ignored) → Code node
   - Leave the **true** output unconnected to stop ignored messages.

6) **Add Code node to extract sender email**
   - Node: **Code**
   - Name: **Code - Extract Sender Email**
   - JavaScript:
     ```js
     const fromRaw = $json.from_raw || $json.from || '';
     const m = fromRaw.match(/<([^>]+)>/);
     const email = (m ? m[1] : fromRaw).trim().toLowerCase();

     return [{
       ...$json,
       sender_email: email,
     }];
     ```
   - Connect: Code → Google Sheets node

7) **Add Google Sheets node to check reply history**
   - Node: **Google Sheets**
   - Name: **Get row(s) in sheet**
   - Credentials: **Google Sheets OAuth2**
   - Operation: **Get row(s)** (read)
   - Document: select your spreadsheet (create one named e.g. `reply_history`)
   - Sheet/tab: select the tab containing your data (in the JSON it is “シート1”)
   - Ensure the sheet has a header column named: **`sender_email`**
   - Filters / Lookup:
     - Lookup column: `sender_email`
     - Lookup value: `={{$json.sender_email}}`
   - Connect: Sheets → IF Found in Reply History?

8) **Add IF node to stop if found in sheet**
   - Node: **IF**
   - Name: **IF - Found in Reply History?**
   - Boolean condition (true/false):
     - Left value: `={{$input.all().length > 0}}`
     - Check: “is true”
   - Connect **false** output (not found) → Gmail search node
   - Leave **true** output unconnected (stop if already replied).

9) **Add Gmail node to search Sent mail**
   - Node: **Gmail**
   - Name: **Gmail - Search Sent to Sender**
   - Credentials: **Gmail OAuth2** (same account as trigger)
   - Operation: **Get Many / GetAll** messages
   - Limit: `20`
   - Query `q`:
     - `={{"in:sent to:" + $json.sender_email + " newer_than:" + $json.importantReplyHistoryDays + "d"}}`
   - Connect: Gmail Search → IF Sent history exists?

10) **Add IF node to stop if sent history exists**
   - Node: **IF**
   - Name: **IF - Sent history exists?**
   - Condition:
     - Left value: `={{$items().length > 0}}`
     - Check: “is true”
   - Connect **false** output → OpenAI “Message a model”
   - Leave **true** output unconnected (stop if duplicate risk).

11) **Add OpenAI node to generate reply**
   - Node: **OpenAI (Message a model)** (the LangChain OpenAI node in n8n)
   - Name: **Message a model**
   - Credentials: **OpenAI API**
   - Model: **gpt-4o-mini**
   - Prompt/message content (expression):
     ```
     Write a polite and natural English email reply to the following message.

     Rules:
     - Do not guess missing information.
     - Be clear and concise.
     - Sound human, not automated.
     - Under 120 words.
     - End with one clear next action.

     Subject:
     {{$json.subject}}

     Email body:
     {{$json.text || $json.snippet}}
     ```
   - Connect: OpenAI node → Reply node

12) **Add Gmail Reply node**
   - Node: **Gmail**
   - Name: **Reply to a message**
   - Credentials: **Gmail OAuth2**
   - Operation: **Reply**
   - Message ID: `={{$json.id}}`
   - Message body: `={{$json.text}}`
   - Email type: **text**
   - Verify that:
     - The OpenAI node output includes the generated reply in `$json.text`
     - The original email `id` is still available at this step (if not, you must merge/keep it)

13) **(Optional) Add sticky notes for maintainability**
   - Add sticky notes matching the five “Step” blocks and the overview (as in the provided workflow) to document intent.

14) **Test with controlled emails**
   - Send yourself a test email (non-promotional subject)
   - Confirm:
     - It is not filtered by ignore rules
     - Sender lookup in Sheets behaves as expected
     - Sent search returns 0 when appropriate
     - Reply is posted in the correct thread

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically replies to important incoming Gmail messages. It avoids duplicate replies by checking both a Google Sheet reply history and recent sent emails in Gmail before responding. Only emails that pass all filters receive an AI-generated English reply.” | Workflow overview sticky note |
| Step 1 note: Email intake, rules, normalization for stable downstream processing | Sticky Note1 |
| Step 2 note: Ignore newsletters/promotions/irrelevant emails; stop immediately when matched | Sticky Note2 |
| Step 3 note: Extract sender email; check Google Sheet reply history; stop if found | Sticky Note3 |
| Step 4 note: Search Gmail sent mail within configured time window; stop if found | Sticky Note4 |
| Step 5 note: Generate English reply with AI; send Gmail reply only if no duplicates detected | Sticky Note5 |
| Disclaimer (FR): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided by requester (global context) |