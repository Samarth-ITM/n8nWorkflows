Classify and auto-reply to Gmail with OpenAI, Google Sheets RAG and Telegram

https://n8nworkflows.xyz/workflows/classify-and-auto-reply-to-gmail-with-openai--google-sheets-rag-and-telegram-11265


# Classify and auto-reply to Gmail with OpenAI, Google Sheets RAG and Telegram

## 1. Workflow Overview

**Title:** Classify and auto-reply to Gmail with OpenAI, Google Sheets RAG and Telegram  
**Purpose:** Monitor incoming **UNREAD Gmail emails**, classify them using **OpenAI**, route them into one of four categories (High Priority, Customer Support, Promotions, Finance/Billing), then **label + process** them (draft, auto-reply, summarize/forward), and optionally **notify teams via Telegram**.  
**Primary use cases:** inbox triage, customer support automation, promotional filtering, finance escalation, and human-in-the-loop handling for urgent emails.

### 1.1 Entry & Classification
- Trigger on new unread Gmail messages
- Use OpenAI to clean/normalize the email content and output a JSON classification
- Switch routes to one of four processing branches

### 1.2 High Priority Handling (Human-in-the-loop)
- Apply Gmail label “High Priority”
- OpenAI generates a **draft** response + internal notification text
- Create a **Gmail Draft** (not sent automatically)
- (Optional/partially disabled) mark as read and remove from inbox
- Telegram notification to stakeholders

### 1.3 Customer Support Auto-Responder (RAG via Google Sheets Tool)
- Apply Gmail label “Customer Support”
- OpenAI writes a reply (with access to a Google Sheets “tool” for internal info)
- Send Gmail **thread reply**
- (Optional/partially disabled) mark read and remove from inbox
- Telegram notification confirming auto-reply

### 1.4 Promotions Handling
- Apply Gmail label “Promotions”
- OpenAI summarizes and recommends actions
- Mark as read, remove from inbox
- Telegram notification

### 1.5 Finance/Billing Handling
- Apply Gmail label “Finance/Billing”
- OpenAI summarizes into Subject/Message + internal notification
- Send an email (as a new outbound email) back to the original sender address (as configured)
- (Optional/partially disabled) mark read, remove from inbox
- Telegram notification

### 1.6 Demo Utilities (Disabled)
- Send demo emails from rows in Google Sheets (bulk testing)
- Clean inbox by deleting messages in batches (dangerous; deletes emails)

---

## 2. Block-by-Block Analysis

### Block A — Gmail Intake & AI Classification
**Overview:** Watches for unread Gmail messages, sends the email content to an OpenAI “classifier agent” that outputs normalized JSON including a `classification`, then routes via a Switch node.  
**Nodes involved:** `Gmail Trigger1`, `Email Classifier Agent`, `Switch`

#### Node: Gmail Trigger1
- **Type/Role:** Gmail Trigger — polls inbox for new matching messages.
- **Config choices:**  
  - Filter: `labelIds = ["UNREAD"]` (only unread messages)  
  - Polling: every minute  
  - `simple=false` (returns a richer message object, including `text`, `subject`, `from`, etc.)
- **Outputs to:** `Email Classifier Agent`
- **Failure modes / edge cases:**  
  - OAuth expiration/insufficient Gmail scopes  
  - Large email bodies/attachments may not be fully represented in `text`  
  - Race condition: messages marked read externally may be missed

#### Node: Email Classifier Agent
- **Type/Role:** OpenAI (LangChain) — classification + normalization.
- **Model:** `gpt-4.1-mini`
- **Key behaviors/config:**  
  - `jsonOutput=true` (expects strict JSON back)  
  - `simplify=false` (keeps full OpenAI response structure; downstream uses `choices[0].message.content...`)  
  - System instructions define 4 categories and require: “Do not explain reasoning. Only output JSON.”
  - Additional assistant instruction: “clean up html tags … output clean data in JSON format … includes `classification` field.”
  - Includes email content via expressions: `{{ $json.subject }}`, `{{ $json.text }}`, and `{{ JSON.stringify($json) }}`
- **Outputs to:** `Switch`
- **Failure modes / edge cases:**  
  - If model returns invalid JSON, Switch expressions will fail (cannot read `classification`)  
  - If output schema differs (e.g., `classification` missing), routing breaks  
  - Email body may include encoded/HTML content; prompt asks to clean, but reliability varies  
  - Token limits for very large emails; `JSON.stringify($json)` can be huge

#### Node: Switch
- **Type/Role:** Switch/router — routes by classification string.
- **Rules (4 outputs):**
  - `High_Priority` when `{{ $json.choices[0].message.content.classification }}` equals `High_Priority`
  - `Customer_Support` equals `Customer_Support`
  - `Promotions` equals `Promotions`
  - `Finance/Billing` equals `Finance/Billing`
- **Connections:**  
  - Output 0 → `High Priority1`  
  - Output 1 → `Customer Support1`  
  - Output 2 → `Promotion1`  
  - Output 3 → `Finance/Billing1`
- **Failure modes / edge cases:**  
  - Exact match required; any variation (“Finance”, “Billing”, casing) won’t route  
  - If OpenAI output is simplified differently, `choices[0]...` path may be wrong  
  - No default/fallback route is defined

---

### Block B — High Priority Branch (Label → Draft → Optional Cleanup → Telegram)
**Overview:** Labels the email as High Priority, uses OpenAI to produce a structured draft response, creates a Gmail draft for human review, then removes it from inbox and notifies via Telegram.  
**Nodes involved:** `High Priority1`, `Creating Draft1`, `Draft1`, `Mark as Read` (disabled), `Remove From Inbox`, `Notify (1)1`

#### Node: High Priority1
- **Type/Role:** Gmail — add label.
- **Config choices:**  
  - Operation: Add Labels  
  - Label: `Label_2067919150642379858` (High Priority label in this account)  
  - Message ID: `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Creating Draft1`
- **Failure modes / edge cases:**  
  - Label ID must exist in the connected Gmail account  
  - If trigger output lacks `id`, expression fails

#### Node: Creating Draft1
- **Type/Role:** OpenAI — generate structured content for a draft + internal message.
- **Model:** `gpt-4o`
- **Config choices:**  
  - System message: executive assistant; includes incoming email text: `{{ $('Gmail Trigger1').item.json.text }}`  
  - Requires JSON output with fields: `Subject`, `Message`, `Email_Draft`, `Text_Message`  
  - `jsonOutput=true`
- **Outputs to:** `Draft1`
- **Failure modes / edge cases:**  
  - If returned JSON doesn’t include `Email_Draft`/`Subject`, Draft creation fails  
  - Hallucinated/unsafe content risk (no explicit guardrails beyond the prompt)

#### Node: Draft1
- **Type/Role:** Gmail — create a draft (human-in-the-loop).
- **Config choices:**  
  - Resource: Draft  
  - Subject: `{{ $json.message.content.Subject }}`  
  - Message: `{{ $json.message.content.Email_Draft }}`
- **Inputs from:** `Creating Draft1`
- **Outputs to:** `Mark as Read` (disabled)
- **Failure modes / edge cases:**  
  - If OpenAI node output path differs, expressions fail (`$json.message.content...`)  
  - Gmail draft creation can fail due to quota/scopes

#### Node: Mark as Read (disabled)
- **Type/Role:** Gmail — mark email as read.
- **Config choices:**  
  - Operation: markAsRead  
  - Message ID expression is likely incorrect: `{{ $json.choices[0].message.content.id }}` (Draft creation output is not OpenAI `choices`; it’s Gmail draft data)
- **Inputs from:** `Draft1`
- **Outputs to:** `Remove From Inbox`
- **Failure modes / edge cases:**  
  - This node is disabled; if enabled, it would likely fail due to wrong messageId mapping

#### Node: Remove From Inbox
- **Type/Role:** Gmail — remove INBOX label (archive).
- **Config choices:**  
  - Operation: removeLabels  
  - labelIds: `["INBOX"]`  
  - messageId: `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Notify (1)1`
- **Failure modes / edge cases:**  
  - Removing INBOX archives; may not be desired for “high priority”  
  - If message already archived, operation may still succeed but is redundant

#### Node: Notify (1)1
- **Type/Role:** Telegram — notify team about high priority handling.
- **Config choices:**  
  - chatId: `123456789`  
  - text: `{{ $('Creating Draft1').item.json.message.content.Text_Message }}`
- **Failure modes / edge cases:**  
  - Invalid chat ID or bot not in chat  
  - Telegram credential/token revoked  
  - If `Text_Message` missing, message text becomes empty/undefined

---

### Block C — Customer Support Branch (Label → RAG Reply → Thread Reply → Optional Cleanup → Telegram)
**Overview:** Labels customer support emails, uses OpenAI to generate a response (with access to Google Sheets as a tool), replies in-thread, then archives and notifies via Telegram.  
**Nodes involved:** `Customer Support1`, `Creating Email1`, `FTAI Info`, `Auto Reply1`, `Mark as Read1` (disabled), `Remove From Inbox1`, `Notify (2)1`

#### Node: Customer Support1
- **Type/Role:** Gmail — add label.
- **Config choices:**  
  - Label: `Label_1598855517228333727`  
  - Message ID: `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Creating Email1`
- **Failure modes:** same as other label nodes (label existence, auth)

#### Node: Creating Email1
- **Type/Role:** OpenAI — generate customer support reply.
- **Model:** `gpt-4o`
- **Config choices:**  
  - System prompt: customer service representative; includes fallback escalation email `yourname@xyz.com`  
  - Injects incoming email text: `{{ $('Gmail Trigger1').item.json.text }}`
  - Uses an **AI Tool** connection to Google Sheets tool node `FTAI Info` (see below)
  - Note: `jsonOutput` is **not enabled** here; output is treated as plain text and referenced as `{{ $json.message.content }}`
- **Outputs to:** `Auto Reply1`
- **Failure modes / edge cases:**  
  - Tool use may fail if Sheets permissions/scopes missing  
  - If OpenAI returns non-string structure unexpectedly, reply expression may misbehave  
  - Prompt relies on “internal Q&A and SOP” but only tool is Sheets; data quality matters

#### Node: FTAI Info
- **Type/Role:** Google Sheets Tool (LangChain tool) — allows the OpenAI node to query a sheet.
- **Config choices:**  
  - Document: “FTAIM.com Outlook Inbox Demo” (spreadsheet ID `1Lz3...`)  
  - Sheet/tab: “FTAI RAG” (gid `1081927447`)
- **Connections:** Tool output is wired to `Creating Email1` via `ai_tool`.
- **Failure modes / edge cases:**  
  - OAuth scope/permission errors  
  - Tool schema mismatch if sheet structure changes  
  - The OpenAI prompt must “decide” to call the tool; not guaranteed unless explicitly instructed (classifier prompt mentions tool access; responder prompt does not)

#### Node: Auto Reply1
- **Type/Role:** Gmail — reply in the same thread.
- **Config choices:**  
  - Resource: Thread, Operation: reply  
  - threadId: `{{ $('Customer Support1').item.json.threadId }}`  
  - messageId: `{{ $('Customer Support1').item.json.id }}`  
  - message body: `{{ $json.message.content }}`
- **Outputs to:** `Mark as Read1` (disabled)
- **Failure modes / edge cases:**  
  - If `threadId` isn’t present in the upstream item, reply fails  
  - Reply may include signatures/quoting depending on Gmail defaults
  - If multiple items are processed, referencing `$('Customer Support1').item...` assumes consistent item pairing

#### Node: Mark as Read1 (disabled)
- **Type/Role:** Gmail — mark message as read.
- **Config:** messageId `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Remove From Inbox1`
- **Notes:** Disabled; if enabled, should work (unlike the High Priority “Mark as Read” node).

#### Node: Remove From Inbox1
- **Type/Role:** Gmail — archive by removing INBOX label.
- **Config:** messageId `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Notify (2)1`

#### Node: Notify (2)1
- **Type/Role:** Telegram — notify support team.
- **Config:** static message: `"customer support query was auto replied"`

---

### Block D — Promotions Branch (Label → Summarize → Mark Read → Archive → Telegram)
**Overview:** Labels promotions, has OpenAI generate a summary + recommendation, then marks read, archives, and notifies via Telegram.  
**Nodes involved:** `Promotion1`, `Promotions`, `Mark as Read2`, `Remove From Inbox2`, `Notify (3)1`

#### Node: Promotion1
- **Type/Role:** Gmail — add Promotions label.
- **Label:** `Label_4204277247566580564`
- **Outputs to:** `Promotions`

#### Node: Promotions
- **Type/Role:** OpenAI — summarize promotional content.
- **Model:** `gpt-4o`
- **Config:** Prompt requests `Summary`, `Recommendation`, `Text_Message` and `jsonOutput=true`.
- **Outputs to:** `Mark as Read2`
- **Edge cases:** missing fields in JSON output; token size issues if email is long.

#### Node: Mark as Read2
- **Type/Role:** Gmail — mark as read.
- **Config:** messageId `{{ $('Gmail Trigger1').item.json.id }}`
- **Outputs to:** `Remove From Inbox2`

#### Node: Remove From Inbox2
- **Type/Role:** Gmail — archive.
- **Config:** remove label `INBOX` for messageId from trigger
- **Outputs to:** `Notify (3)1`

#### Node: Notify (3)1
- **Type/Role:** Telegram — notify.
- **Config:** `"Promotional email removed from the Inbox"`

---

### Block E — Finance/Billing Branch (Label → Summarize → Send Email → Optional Cleanup → Telegram)
**Overview:** Labels finance/billing emails, summarizes them via OpenAI, then sends an outbound email (not a thread reply) to the original sender address, and archives + notifies.  
**Nodes involved:** `Finance/Billing1`, `Finance / Billing Dept`, `Send to Finance Dept1`, `Mark as Read3` (disabled), `Remove From Inbox3`, `Notify (4)1`

#### Node: Finance/Billing1
- **Type/Role:** Gmail — add Finance/Billing label.
- **Label:** `Label_6542868233920128707`
- **Outputs to:** `Finance / Billing Dept`

#### Node: Finance / Billing Dept
- **Type/Role:** OpenAI — summarize billing email for internal handling.
- **Model:** `gpt-4o`
- **Config:** `jsonOutput=true` requiring `Subject`, `Message`, `Text_Message`.
- **Outputs to:** `Send to Finance Dept1`

#### Node: Send to Finance Dept1
- **Type/Role:** Gmail — send an outbound email.
- **Config choices:**  
  - `sendTo`: `{{ $('Gmail Trigger1').item.json.from.value[0].address }}` (sends to original sender)  
  - `subject`: `{{ $json.message.content.Subject }}`  
  - `message`: `{{ $json.message.content.Message }}`  
  - `appendAttribution=false`  
  - `emailType=text`
- **Outputs to:** `Mark as Read3` (disabled)
- **Failure modes / edge cases:**  
  - **Data shape risk:** Gmail Trigger’s `from` structure often differs from Outlook-style (`from.value[0].address` looks like Microsoft Graph). If Gmail trigger returns `{ from: { value: ... } }` is not guaranteed. This may be a leftover from another workflow and can break.  
  - Potentially sends sensitive summaries to external sender (not internal finance). If the intent was to forward internally, `sendTo` should be finance team address instead.  
  - Subject/Message missing -> send fails

#### Node: Mark as Read3 (disabled)
- **Type/Role:** Gmail — mark as read.
- **Config:** messageId from trigger
- **Outputs to:** `Remove From Inbox3`

#### Node: Remove From Inbox3
- **Type/Role:** Gmail — archive.
- **Outputs to:** `Notify (4)1`

#### Node: Notify (4)1
- **Type/Role:** Telegram — notify finance urgency.
- **Config:** `"There is a billing enquiry that needs your immediate attention"`

---

### Block F — Demo Email Generator (Disabled)
**Overview:** Manually triggered flow that reads rows from a Google Sheet (“Demo_Emails”) and sends them as emails to test classification.  
**Nodes involved:** `Sticky Note2` (disabled), `When clicking ‘Execute workflow’` (disabled), `Get row(s) in sheet` (disabled), `Send Demo Emails` (disabled)

#### Node: When clicking ‘Execute workflow’ (disabled)
- **Type/Role:** Manual Trigger — start testing flow.
- **Outputs to:** `Get row(s) in sheet`

#### Node: Get row(s) in sheet (disabled)
- **Type/Role:** Google Sheets — read demo email rows.
- **Config:** reads from Spreadsheet `1Lz3...`, Sheet “Demo_Emails” (gid `1833398336`)
- **Outputs to:** `Send Demo Emails`
- **Failure modes:** permissions, changed columns, empty rows.

#### Node: Send Demo Emails (disabled)
- **Type/Role:** Gmail — send emails from sheet content.
- **Config:**  
  - `sendTo: user@example.com` (must be changed to your test inbox)  
  - subject: `{{ $json.Subject }}`  
  - message: `{{ $json.Body }}`
- **Failure modes:** quota, invalid recipient, sheet missing fields.

#### Sticky Note2 (disabled)
- Comment: “Demo Email Generator — Sends sample emails to test classification logic.”

---

### Block G — Demo Cleanup / Delete Emails (Disabled)
**Overview:** Deletes emails in batches to reset inbox after testing, then sends a Telegram confirmation.  
**Nodes involved:** `Sticky Note3` (disabled), `Read_all_Emails` (disabled), `Loop Over Items` (disabled), `Delete a message` (disabled), `Clean Inbox` (disabled), `Sticky Note5`

#### Node: Read_all_Emails (disabled)
- **Type/Role:** Gmail — get all messages.
- **Config:** `includeSpamTrash=true`, `returnAll=true`
- **Outputs to:** `Loop Over Items`
- **Risk:** pulls everything; can delete real emails downstream.

#### Node: Loop Over Items (disabled)
- **Type/Role:** SplitInBatches — batch size 5.
- **Connections:**  
  - Output 0 → `Clean Inbox` and `Delete a message`  
  - `Delete a message` loops back to `Loop Over Items` (classic batch deletion loop)
- **Edge cases:** if email list is huge, long runtime; risk hitting API limits.

#### Node: Delete a message (disabled)
- **Type/Role:** Gmail — delete message by `{{ $json.id }}`
- **Outputs to:** loops back to `Loop Over Items`
- **Risk:** permanent deletion depending on Gmail behavior/config; dangerous with real account.

#### Node: Clean Inbox (disabled, executeOnce=true)
- **Type/Role:** Telegram — “All Demo Emails Deleted”
- **Note:** wired in a way that may send early; but `executeOnce=true` prevents spam.

#### Sticky Note3 (disabled)
- Comment: “Demo Cleanup — Deletes test emails to reset the inbox.”

#### Sticky Note5
- Comment: “Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully)… Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger1 | Gmail Trigger | Poll unread emails | — | Email Classifier Agent | # 📨 Gmail Classifier —Classify emails using AI and automate responses / How it works / Setup steps / Author + links |
| Email Classifier Agent | OpenAI (LangChain) | Normalize + classify email into 4 categories | Gmail Trigger1 | Switch | # 📨 Gmail Classifier —Classify emails using AI and automate responses / How it works / Setup steps / Author + links |
| Switch | Switch | Route by classification | Email Classifier Agent | High Priority1; Customer Support1; Promotion1; Finance/Billing1 | # 📨 Gmail Classifier —Classify emails using AI and automate responses / How it works / Setup steps / Author + links |
| High Priority1 | Gmail | Add High Priority label | Switch | Creating Draft1 | ## High Priority Emails / A Draft will be created for Human in the Loop |
| Creating Draft1 | OpenAI (LangChain) | Generate structured draft + internal message | High Priority1 | Draft1 | ## High Priority Emails / A Draft will be created for Human in the Loop |
| Draft1 | Gmail | Create Gmail draft (human review) | Creating Draft1 | Mark as Read | ## Human in the Loop Review / **Before sending out |
| Mark as Read | Gmail | Mark email read (disabled; misconfigured id source) | Draft1 | Remove From Inbox | ## Human in the Loop Review / **Before sending out |
| Remove From Inbox | Gmail | Archive (remove INBOX label) | Mark as Read | Notify (1)1 | ## Human in the Loop Review / **Before sending out |
| Notify (1)1 | Telegram | Notify team about high priority | Remove From Inbox | — | ## Human in the Loop Review / **Before sending out |
| Customer Support1 | Gmail | Add Customer Support label | Switch | Creating Email1 | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Creating Email1 | OpenAI (LangChain) | Write support reply (can use Sheets tool) | Customer Support1 | Auto Reply1 | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| FTAI Info | Google Sheets Tool | AI tool for course/company info (RAG) | — (tool link) | Creating Email1 (ai_tool) | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Auto Reply1 | Gmail | Reply in thread | Creating Email1 | Mark as Read1 | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Mark as Read1 | Gmail | Mark as read (disabled) | Auto Reply1 | Remove From Inbox1 | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Remove From Inbox1 | Gmail | Archive | Mark as Read1 | Notify (2)1 | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Notify (2)1 | Telegram | Notify auto-replied | Remove From Inbox1 | — | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Promotion1 | Gmail | Add Promotions label | Switch | Promotions |  |
| Promotions | OpenAI (LangChain) | Summarize + recommend | Promotion1 | Mark as Read2 |  |
| Mark as Read2 | Gmail | Mark as read | Promotions | Remove From Inbox2 |  |
| Remove From Inbox2 | Gmail | Archive | Mark as Read2 | Notify (3)1 |  |
| Notify (3)1 | Telegram | Notify promotions archived | Remove From Inbox2 | — |  |
| Finance/Billing1 | Gmail | Add Finance/Billing label | Switch | Finance / Billing Dept |  |
| Finance / Billing Dept | OpenAI (LangChain) | Summarize billing request | Finance/Billing1 | Send to Finance Dept1 |  |
| Send to Finance Dept1 | Gmail | Send outbound email (currently to original sender) | Finance / Billing Dept | Mark as Read3 |  |
| Mark as Read3 | Gmail | Mark as read (disabled) | Send to Finance Dept1 | Remove From Inbox3 |  |
| Remove From Inbox3 | Gmail | Archive | Mark as Read3 | Notify (4)1 |  |
| Notify (4)1 | Telegram | Notify finance urgency | Remove From Inbox3 | — |  |
| When clicking ‘Execute workflow’ | Manual Trigger | Start demo email generator (disabled) | — | Get row(s) in sheet | ## Demo Email Generator / Sends sample emails to test classification logic. |
| Get row(s) in sheet | Google Sheets | Load demo emails from sheet (disabled) | When clicking ‘Execute workflow’ | Send Demo Emails | ## 📤 Send Demo Emails (Testing and Simulating bulk email) / What This Does / Why Use It |
| Send Demo Emails | Gmail | Send demo emails (disabled) | Get row(s) in sheet | — | ## 📤 Send Demo Emails (Testing and Simulating bulk email) / What This Does / Why Use It |
| Read_all_Emails | Gmail | Fetch all emails for cleanup (disabled) | — | Loop Over Items | ## 🗑️ Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully) / Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT. |
| Loop Over Items | SplitInBatches | Batch deletion loop (disabled) | Read_all_Emails; Delete a message | Clean Inbox; Delete a message | ## 🗑️ Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully) / Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT. |
| Delete a message | Gmail | Delete email by id (disabled) | Loop Over Items | Loop Over Items | ## 🗑️ Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully) / Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT. |
| Clean Inbox | Telegram | Confirm cleanup (disabled) | Loop Over Items | — | ## 🗑️ Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully) / Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT. |
| Sticky Note | Sticky Note | Workflow description + setup + author links | — | — | # 📨 Gmail Classifier —Classify emails using AI and automate responses / **Community:** https://www.skool.com/aic-plus / **Website:** https://www.fasttrackaimastery.com |
| Sticky Note2 | Sticky Note | Demo generator note (disabled) | — | — | ## Demo Email Generator / Sends sample emails to test classification logic. |
| Sticky Note3 | Sticky Note | Demo cleanup note (disabled) | — | — | ## Demo Cleanup / Deletes test emails to reset the inbox. |
| Sticky Note4 | Sticky Note | Send demo emails note | — | — | ## 📤 Send Demo Emails (Testing and Simulating bulk email) / What This Does / Why Use It |
| Sticky Note5 | Sticky Note | Delete demo emails warning | — | — | ## 🗑️ Delete Demo Emails (⚠️ Wipes Inbox — Use Carefully) / Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT. |
| Sticky Note6 | Sticky Note | High priority note | — | — | ## High Priority Emails / A Draft will be created for Human in the Loop |
| Sticky Note7 | Sticky Note | Human review note | — | — | ## Human in the Loop Review / **Before sending out |
| Sticky Note9 | Sticky Note | Support responder note | — | — | ## Customer Support Auto-Responder / AI generates replies using internal Q&A and SOP data. |
| Sticky Note12 | Sticky Note | Demo section header | — | — | ## Demo Email setup and Cleanup ( Activate if Needed) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Gmail OAuth2 credential (scopes to read, modify labels, create drafts, send/reply).  
   2. OpenAI API credential.  
   3. (Optional) Telegram Bot credential (BotFather token) and ensure the bot is in the target chat.  
   4. (Optional) Google Sheets OAuth2 credential with access to the target spreadsheet.

2) **Add Trigger**
   1. Add **Gmail Trigger** node named `Gmail Trigger1`.  
   2. Configure: Filters → Label IDs = `UNREAD`; Polling = every minute; `simple=false`.

3) **Add AI Classifier**
   1. Add **OpenAI (LangChain)** node named `Email Classifier Agent`.  
   2. Model: `gpt-4.1-mini`.  
   3. Enable **JSON Output**.  
   4. Add messages:
      - User message including `subject` + `text` from the trigger.
      - System message describing the four categories and requiring JSON only.
      - Additional instruction to “clean HTML tags” and output normalized JSON including `classification`.
   5. Connect: `Gmail Trigger1` → `Email Classifier Agent`.

4) **Add Switch Router**
   1. Add **Switch** node named `Switch`.  
   2. Create 4 rules with equals comparisons against:  
      `{{ $json.choices[0].message.content.classification }}`  
      matching exactly: `High_Priority`, `Customer_Support`, `Promotions`, `Finance/Billing`.  
   3. Connect: `Email Classifier Agent` → `Switch`.

5) **High Priority branch**
   1. Add **Gmail** node `High Priority1`: Operation “Add Labels”, label = your High Priority label ID, messageId = `{{ $('Gmail Trigger1').item.json.id }}`.  
   2. Add **OpenAI** node `Creating Draft1` (model `gpt-4o`, JSON output on) prompting for `Subject`, `Message`, `Email_Draft`, `Text_Message`.  
   3. Add **Gmail** node `Draft1`: Resource “Draft”, Subject = `{{ $json.message.content.Subject }}`, Message = `{{ $json.message.content.Email_Draft }}`.  
   4. Add **Gmail** node `Remove From Inbox`: Operation “Remove Labels”, labelIds includes `INBOX`, messageId from trigger.  
   5. Add **Telegram** node `Notify (1)1`: text = `{{ $('Creating Draft1').item.json.message.content.Text_Message }}`, chatId set.  
   6. Connect Switch output `High_Priority` → `High Priority1` → `Creating Draft1` → `Draft1`.  
   7. (Optional) If you truly want mark-as-read here, add/enable a correct **Mark as read** node using trigger messageId, then connect to archive + notify.

6) **Customer Support branch (with Sheets tool)**
   1. Add **Gmail** node `Customer Support1`: Add Labels with your support label ID.  
   2. Add **Google Sheets Tool** node `FTAI Info`: select Spreadsheet + “FTAI RAG” tab.  
   3. Add **OpenAI** node `Creating Email1` (model `gpt-4o`): system prompt as customer service agent; include email text. Keep it as plain text output (no JSON required).  
   4. Connect `FTAI Info` to `Creating Email1` using the **AI Tool** connection type.  
   5. Add **Gmail** node `Auto Reply1`: Operation “Reply” (thread). Set `threadId` and `messageId` from the labeled email item, message body from `{{ $json.message.content }}`.  
   6. Add **Gmail** node `Remove From Inbox1` (remove `INBOX`).  
   7. Add **Telegram** node `Notify (2)1` with a fixed message.  
   8. Connect Switch output `Customer_Support` → `Customer Support1` → `Creating Email1` → `Auto Reply1` → (optional mark read) → `Remove From Inbox1` → `Notify (2)1`.

7) **Promotions branch**
   1. Add **Gmail** node `Promotion1` to add the Promotions label ID.  
   2. Add **OpenAI** node `Promotions` (model `gpt-4o`, JSON output on) requesting Summary/Recommendation/Text_Message.  
   3. Add **Gmail** node `Mark as Read2` (markAsRead, messageId from trigger).  
   4. Add **Gmail** node `Remove From Inbox2` (remove `INBOX`).  
   5. Add **Telegram** node `Notify (3)1` with fixed message.  
   6. Connect Switch output `Promotions` → `Promotion1` → `Promotions` → `Mark as Read2` → `Remove From Inbox2` → `Notify (3)1`.

8) **Finance/Billing branch**
   1. Add **Gmail** node `Finance/Billing1` to add finance label ID.  
   2. Add **OpenAI** node `Finance / Billing Dept` (model `gpt-4o`, JSON output on) requesting Subject/Message/Text_Message.  
   3. Add **Gmail** node `Send to Finance Dept1` to send an email using Subject/Message from the OpenAI JSON.  
      - Important: decide the correct recipient. The provided workflow uses `from.value[0].address` which may not exist in Gmail trigger output and also sends externally.
   4. Add **Gmail** node `Remove From Inbox3` (remove `INBOX`).  
   5. Add **Telegram** node `Notify (4)1`.  
   6. Connect Switch output `Finance/Billing` → `Finance/Billing1` → `Finance / Billing Dept` → `Send to Finance Dept1` → (optional mark read) → `Remove From Inbox3` → `Notify (4)1`.

9) **(Optional) Demo generator (disabled in original)**
   1. Add **Manual Trigger**.  
   2. Add **Google Sheets** “Get row(s)” reading your demo email rows (must contain `Subject` and `Body`).  
   3. Add **Gmail Send** node to email those rows to your test inbox.  
   4. Keep the branch disabled unless testing.

10) **(Optional) Demo cleanup deletion loop (disabled in original)**
   1. Add Gmail “Get All” messages node (very risky).  
   2. Add SplitInBatches node and Gmail “Delete” node looping until done.  
   3. Add Telegram confirmation.  
   4. Only enable with a dedicated test account/mailbox.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: Sandeep Patharkar | From workflow sticky note |
| Community | https://www.skool.com/aic-plus |
| Website | https://www.fasttrackaimastery.com |
| Demo Email Generator sends sample emails to test classification logic | Included but disabled |
| Delete Demo Emails warning: “Wipes Inbox — Use Carefully… Be VERY CAREFUL WITH THE REAL GMAIL ACCOUNT.” | Demo cleanup block |
| High Priority: “A Draft will be created for Human in the Loop” + “Before sending out” | Draft review process notes |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.