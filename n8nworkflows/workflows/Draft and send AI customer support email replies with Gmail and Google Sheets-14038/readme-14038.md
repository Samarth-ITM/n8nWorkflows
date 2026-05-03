Draft and send AI customer support email replies with Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/draft-and-send-ai-customer-support-email-replies-with-gmail-and-google-sheets-14038


# Draft and send AI customer support email replies with Gmail and Google Sheets

# 1. Workflow Overview

This workflow automates a human-in-the-loop customer support email process using Gmail, OpenRouter-based AI drafting, and Google Sheets as a review queue.

Its purpose is to:
- monitor incoming Gmail messages,
- ignore likely automated or irrelevant emails,
- generate a draft support reply with AI,
- log the original message and proposed reply into Google Sheets,
- allow a human reviewer to approve sending by typing `send` in a spreadsheet column,
- then send the approved reply and mark the row as completed.

The workflow has **two entry points** and is best understood in two functional blocks.

## 1.1 Email Intake, Filtering, AI Drafting, and Logging

This block starts when a new Gmail message is detected. It filters out likely auto-generated or unwanted messages, sends the email content to an LLM via OpenRouter, marks the original email as read, and stores the original email plus AI-generated draft in Google Sheets for human review.

## 1.2 Review Queue Polling, Approval Check, Sending, and Status Update

This block runs on a schedule every minute. It looks for spreadsheet rows whose `Send` column contains `send`, confirms approval, sends the drafted reply through Gmail, and updates the corresponding sheet row to `Replied` so it is not sent again.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Email Intake, Filtering, AI Drafting, and Logging

### Overview

This block receives incoming Gmail messages and applies basic anti-noise logic to prevent drafting replies for self-sent, no-reply, newsletter, bounce, or out-of-office emails. Valid customer messages are then passed to an AI model for drafting, after which the email is marked as read and logged into Google Sheets for manual review.

### Nodes Involved

- Receive Customer Emails
- Filter Unwanted Emails
- OpenRouter Model
- Draft AI Reply
- Mark Email as Read
- Log to Google Sheets

---

### Node: Receive Customer Emails

- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Trigger node that polls Gmail for new emails.
- **Configuration choices:**
  - Uses Gmail trigger polling rather than webhook-style push.
  - Polls **every minute**.
  - `simple: false`, so the workflow receives fuller Gmail payload data rather than simplified output.
  - No explicit filters are configured, so all incoming messages are candidates.
- **Key expressions or variables used:**
  - Downstream nodes reference fields such as:
    - `{{$json.id}}`
    - `{{$json.subject}}`
    - `{{$json.text}}`
    - `{{$json.from.value[0].address}}`
    - `{{$json.from}}`
- **Input and output connections:**
  - Entry point node, no input.
  - Outputs to **Filter Unwanted Emails**.
- **Version-specific requirements:**
  - Type version `1.3`.
  - Requires valid Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Gmail OAuth token expiration or revoked access.
  - Polling delay means emails are not processed instantly.
  - If incoming email structure differs, expressions like `from.value[0].address` may fail.
  - Missing plaintext body may reduce usefulness of `$json.text`.
- **Sub-workflow reference:** None.

---

### Node: Filter Unwanted Emails

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional filter that allows only likely real customer emails through the true branch.
- **Configuration choices:**
  - Uses an **AND** combinator.
  - Email proceeds only if all exclusion checks pass.
  - Filters out:
    - a specific own email address: `user@example.com`
    - senders containing `noreply`, `no-reply`, `mailer-daemon`
    - subjects containing:
      - `out of office`
      - `auto-reply`
      - `automatic reply`
      - `delivery notification`
      - `undeliverable`
      - `newsletter`
      - `unsubscribe`
- **Key expressions or variables used:**
  - `={{ $json.from.value[0].address }}`
  - `={{ $json.subject }}`
- **Input and output connections:**
  - Input from **Receive Customer Emails**
  - True output to **Draft AI Reply**
  - False output unused
- **Version-specific requirements:**
  - Type version `2.3`
  - Condition options use IF node condition version `3`
- **Edge cases or potential failure types:**
  - Filtering is case-sensitive by configuration; uppercase or differently formatted subjects may bypass or fail exclusions.
  - If `from.value[0].address` is missing, expression errors may occur.
  - False positives: legitimate emails mentioning words like “unsubscribe” or “newsletter”.
  - False negatives: automated emails with subjects not covered by these phrases.
- **Sub-workflow reference:** None.

---

### Node: OpenRouter Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  AI chat model connector used as the language model backend for the drafting chain.
- **Configuration choices:**
  - Minimal options configured.
  - Relies primarily on OpenRouter credentials and whichever default model/settings are defined in the credential or node UI at runtime.
- **Key expressions or variables used:**
  - None in the node itself.
- **Input and output connections:**
  - Connected via `ai_languageModel` to **Draft AI Reply**
  - Does not participate in the main execution chain directly.
- **Version-specific requirements:**
  - Type version `1`
  - Requires OpenRouter API credentials.
- **Edge cases or potential failure types:**
  - Missing API key, invalid API key, quota exhaustion, or model access restrictions.
  - Rate limiting or timeout from OpenRouter.
  - If no model is selected in the actual UI, execution may fail depending on node defaults.
- **Sub-workflow reference:** None.

---

### Node: Draft AI Reply

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  LangChain LLM chain node that prepares a structured prompt and asks the connected model to generate a customer support draft reply.
- **Configuration choices:**
  - Prompt type is `define`.
  - Input text sent to the model is assembled from the received email:
    - sender
    - subject
    - message body
  - A single instruction message acts as the system-style guidance:
    - support role for “Your Company Name Here”
    - embedded company documentation
    - instruction not to invent facts
    - instruction to escalate politely if unsure
    - sign-off as `"Support Team"`
    - output only the email body, no subject line
- **Key expressions or variables used:**
  - `{{ $('Receive Customer Emails').item.json.from }}`
  - `{{ $('Receive Customer Emails').item.json.subject }}`
  - `{{ $('Receive Customer Emails').item.json.text }}`
- **Input and output connections:**
  - Main input from **Filter Unwanted Emails**
  - AI language model input from **OpenRouter Model**
  - Main output to **Mark Email as Read**
- **Version-specific requirements:**
  - Type version `1.9`
  - Requires compatible LangChain/OpenRouter node support in the n8n instance.
- **Edge cases or potential failure types:**
  - If the email body is empty or malformed, the draft may be poor or blank.
  - If prompt content exceeds model context limits, request may fail or truncate.
  - Hallucination risk is reduced but not eliminated.
  - Cross-node reference to `Receive Customer Emails` assumes item linkage remains valid.
- **Sub-workflow reference:** None.

---

### Node: Mark Email as Read

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Gmail action node that marks the original incoming message as read.
- **Configuration choices:**
  - Operation: `markAsRead`
  - Uses the Gmail message ID from the trigger payload.
- **Key expressions or variables used:**
  - `={{ $('Receive Customer Emails').item.json.id }}`
- **Input and output connections:**
  - Input from **Draft AI Reply**
  - Output to **Log to Google Sheets**
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials with sufficient mailbox access.
- **Edge cases or potential failure types:**
  - If the message ID is invalid or the message is no longer accessible, operation fails.
  - If draft generation succeeds but Gmail update fails, the message remains unread while downstream logging may stop.
- **Sub-workflow reference:** None.

---

### Node: Log to Google Sheets

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a review record into a Google Sheet containing the original message and AI draft.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `Customer Support Emails`
  - Sheet/tab: `Sheet1`
  - Mapping mode: manually defined columns
  - Writes these fields:
    - `Message ID`
    - `From`
    - `Subject`
    - `Body`
    - `Reply`
  - `Send` column exists in schema but is not populated initially; it is intended for human review input.
- **Key expressions or variables used:**
  - `={{ $('Receive Customer Emails').item.json.text }}`
  - `={{ $('Receive Customer Emails').item.json.from.value[0].address }}`
  - `={{ $('Draft AI Reply').item.json.text }}`
  - `={{ $('Receive Customer Emails').item.json.subject }}`
  - `={{ $('Receive Customer Emails').item.json.id }}`
- **Input and output connections:**
  - Input from **Mark Email as Read**
  - No downstream main output connected
- **Version-specific requirements:**
  - Type version `4.7`
  - Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Spreadsheet or tab not found.
  - Column names in the actual sheet do not match configured schema.
  - Permission issues on the spreadsheet.
  - If duplicate rows are appended for the same message ID, later approval processing may become ambiguous.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Review Queue Polling, Approval Check, Sending, and Status Update

### Overview

This block periodically scans Google Sheets for entries marked `send` in the `Send` column. Approved rows are sent to customers through Gmail, and the corresponding spreadsheet records are updated to `Replied` using `Message ID` as the matching key.

### Nodes Involved

- Check Pending Replies
- Get Approved Replies
- Is Approved to Send?
- Send Reply to Customer
- Update Status to Replied

---

### Node: Check Pending Replies

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Scheduled entry point that periodically checks the spreadsheet review queue.
- **Configuration choices:**
  - Runs every **1 minute**.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Entry point node, no input.
  - Outputs to **Get Approved Replies**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Workflow must be active for scheduled execution.
  - High-frequency polling may create unnecessary load on large sheets.
- **Sub-workflow reference:** None.

---

### Node: Get Approved Replies

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads spreadsheet rows filtered to only those approved for sending.
- **Configuration choices:**
  - Spreadsheet: `Customer Support Emails`
  - Sheet/tab: `Sheet1`
  - Uses a filter:
    - column `Send`
    - value `send`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input from **Check Pending Replies**
  - Output to **Is Approved to Send?**
- **Version-specific requirements:**
  - Type version `4.7`
  - Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - If sheet columns differ in spelling or capitalization, filter may return nothing.
  - Depending on sheet data formatting, `send` with trailing spaces or different case may be missed.
  - Large sheets may slow execution.
- **Sub-workflow reference:** None.

---

### Node: Is Approved to Send?

- **Type and technical role:** `n8n-nodes-base.if`  
  Secondary validation that confirms the row’s `Send` field exactly equals `send`.
- **Configuration choices:**
  - Uses an **OR** combinator with one condition only.
  - Condition:
    - `{{$json.Send}}` equals `send`
- **Key expressions or variables used:**
  - `={{ $json.Send }}`
- **Input and output connections:**
  - Input from **Get Approved Replies**
  - True output to **Send Reply to Customer**
  - False output unused
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - Somewhat redundant because the sheet retrieval already filters for `send`.
  - Case-sensitive equality means `Send`, `SEND`, or `send ` will fail.
- **Sub-workflow reference:** None.

---

### Node: Send Reply to Customer

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the approved reply email to the customer.
- **Configuration choices:**
  - Sends to the address stored in `From`
  - Subject is prefixed with `Re: `
  - Body uses the stored draft reply
  - Replaces newline characters with `<br>` to produce HTML-like line breaks
  - Disables Gmail attribution/footer appending
- **Key expressions or variables used:**
  - `={{ $json.From }}`
  - `={{ $json.Reply.replace(/\n/g, '<br>') }}`
  - `=Re: {{ $json.Subject }}`
- **Input and output connections:**
  - Input from **Is Approved to Send?**
  - Output to **Update Status to Replied**
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - If `From` is not a valid recipient address, send fails.
  - The workflow does not thread via Gmail message/thread ID; it only uses a reply-style subject.
  - HTML formatting assumptions may not match actual message mode if Gmail node treats content differently.
  - If `Reply` is empty, a blank email may be sent unless prevented elsewhere.
- **Sub-workflow reference:** None.

---

### Node: Update Status to Replied

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the Google Sheet row after sending so the same reply is not re-sent.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `Message ID`
  - Writes:
    - `Message ID`
    - `Send = Replied`
  - Spreadsheet: `Customer Support Emails`
  - Sheet/tab: `Sheet1`
- **Key expressions or variables used:**
  - `={{ $('Is Approved to Send?').item.json['Message ID'] }}`
- **Input and output connections:**
  - Input from **Send Reply to Customer**
  - No downstream output connected
- **Version-specific requirements:**
  - Type version `4.7`
  - Requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - If `Message ID` is missing or duplicated in the sheet, updates may target the wrong row or create another row.
  - `appendOrUpdate` behavior depends on exact column matching and sheet schema consistency.
  - If Gmail send succeeds but sheet update fails, the row may remain `send` and could be sent again in a later run.
- **Sub-workflow reference:** None.

---

## 2.3 Documentation and Visual Annotation Nodes

These nodes do not affect execution logic but are important for maintenance and reproduction because they explain setup and usage context.

### Nodes Involved

- Overview
- Section - Email Intake
- Section - Review and Send

---

### Node: Overview

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note describing the workflow purpose, setup, and operational model.
- **Configuration choices:**
  - Large note covering the overall canvas.
  - Includes setup instructions for Gmail, OpenRouter, and Google Sheets.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

### Node: Section - Email Intake

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual section header for the first processing block.
- **Configuration choices:**
  - Labels block ① as email intake and AI draft generation.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

### Node: Section - Review and Send

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual section header for the approval and send block.
- **Configuration choices:**
  - Labels block ② as review and send.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Customer Emails | Gmail Trigger | Poll Gmail every minute for incoming emails |  | Filter Unwanted Emails | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Filter Unwanted Emails | IF | Exclude self-sent, automated, bounce, newsletter, and out-of-office emails | Receive Customer Emails | Draft AI Reply | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Draft AI Reply | LangChain LLM Chain | Build prompt from email content and generate a support reply draft | Filter Unwanted Emails; OpenRouter Model (AI model connection) | Mark Email as Read | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| OpenRouter Model | OpenRouter Chat Model | Provide the LLM backend for the drafting chain |  | Draft AI Reply | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Mark Email as Read | Gmail | Mark processed incoming email as read | Draft AI Reply | Log to Google Sheets | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Log to Google Sheets | Google Sheets | Append original email and AI draft to review sheet | Mark Email as Read |  | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Check Pending Replies | Schedule Trigger | Poll review sheet every minute for approved rows |  | Get Approved Replies | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |
| Get Approved Replies | Google Sheets | Retrieve rows whose Send column is marked `send` | Check Pending Replies | Is Approved to Send? | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |
| Is Approved to Send? | IF | Confirm approval before sending | Get Approved Replies | Send Reply to Customer | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |
| Send Reply to Customer | Gmail | Send approved reply email to customer | Is Approved to Send? | Update Status to Replied | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |
| Update Status to Replied | Google Sheets | Mark approved row as replied using Message ID matching | Send Reply to Customer |  | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |
| Overview | Sticky Note | Global workflow documentation |  |  | ## 📧 AI Customer Support Email Agent<br><br>**Automatically draft AI-powered replies to customer emails, let your team review them in Google Sheets, and send with a single word - keeping a human in the loop.**<br><br>---<br><br>### Who is this for<br>- Small business owners handling customer support manually<br>- Support teams that want AI assistance without losing human oversight<br>- Freelancers and agencies managing client email inquiries<br><br>---<br><br>### How it works<br>1. **Gmail** is monitored every minute for new incoming emails<br>2. **Smart filtering** skips automated emails, newsletters, noreply senders, and out-of-office replies<br>3. **AI (via OpenRouter)** drafts a reply using your company documentation as context<br>4. The email and draft reply are **logged to Google Sheets** for review<br>5. Your team **reviews the draft** - type "send" in the Send column to approve<br>6. Every minute, the workflow **detects approved rows** and sends the reply via Gmail<br>7. The row is **updated to "Replied"** to prevent duplicate sends<br><br>---<br><br>### Knowledge base<br>Your company documentation is embedded **directly in the AI system prompt** - no vector database, no RAG pipeline, no extra infrastructure.<br><br>**Advantages:**<br>- Zero-setup knowledge base - paste your docs and go<br>- Works with any OpenRouter model out of the box<br>- Instant updates - edit the prompt, changes apply immediately<br>- No hallucinations beyond what's in the prompt (the AI is instructed to escalate if unsure)<br><br>**Practical limits:**<br>- Up to ~10 pages (~3,000 words) works reliably with any model<br>- Up to 50+ pages with large-context models (GPT-4 Turbo, Claude 3.5 Sonnet, Llama 3.1 70B and newer versions)<br>- Beyond that, consider a RAG-based approach with a vector store<br><br>---<br><br>### Setup<br>1. **Gmail** - Connect your Gmail OAuth2 account to the Receive Customer Emails, Mark Email as Read, and Send Reply to Customer nodes<br>2. **OpenRouter** - Add your OpenRouter API key to the OpenRouter Model node<br>3. **Google Sheets** - Connect your Google Sheets OAuth2 account to all sheet nodes; create a sheet with columns: Message ID, From, Subject, Body, Reply, Send<br>4. In the **Draft AI Reply** node, update the system prompt with your own company documentation and business name<br>5. In the **Filter Unwanted Emails** node, update the first condition to exclude your own email address<br>6. Activate the workflow |
| Section - Email Intake | Sticky Note | Section header for intake/drafting block |  |  | ## ① Email Intake & AI Draft<br>Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review. |
| Section - Review and Send | Sticky Note | Section header for approval/sending block |  |  | ## ② Review & Send Approved Replies<br>Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer. |

---

# 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow manually in n8n.

## Preparation

1. **Create required credentials**
   1. Add a **Gmail OAuth2** credential in n8n.
   2. Add a **Google Sheets OAuth2** credential in n8n.
   3. Add an **OpenRouter API** credential for the OpenRouter model node.

2. **Create the Google Sheet**
   1. Create a spreadsheet, for example: `Customer Support Emails`.
   2. In the first tab, add these exact column headers:
      - `Message ID`
      - `From`
      - `Subject`
      - `Body`
      - `Reply`
      - `Send`
   3. Ensure the Google Sheets credential has edit access.

3. **Decide your company prompt content**
   - Prepare the company name and support documentation text you want embedded in the AI instruction prompt.
   - Keep it concise enough for your chosen OpenRouter model context window.

---

## Build Block 1: Intake and Drafting

4. **Add a Gmail Trigger node**
   - Name it **Receive Customer Emails**
   - Type: **Gmail Trigger**
   - Set polling to **every minute**
   - Disable simplified output or set `Simple` to **false**
   - Connect your **Gmail OAuth2** credential

5. **Add an IF node**
   - Name it **Filter Unwanted Emails**
   - Connect **Receive Customer Emails → Filter Unwanted Emails**
   - Configure it with **AND** logic
   - Add these conditions using the sender address and subject:
     - sender does **not contain** your own email, e.g. `user@example.com`
     - sender does **not contain** `noreply`
     - sender does **not contain** `no-reply`
     - sender does **not contain** `mailer-daemon`
     - subject does **not contain** `out of office`
     - subject does **not contain** `auto-reply`
     - subject does **not contain** `automatic reply`
     - subject does **not contain** `delivery notification`
     - subject does **not contain** `undeliverable`
     - subject does **not contain** `newsletter`
     - subject does **not contain** `unsubscribe`
   - Use expressions similar to:
     - sender: `{{$json.from.value[0].address}}`
     - subject: `{{$json.subject}}`

6. **Add an OpenRouter chat model node**
   - Name it **OpenRouter Model**
   - Type: **OpenRouter Chat Model**
   - Connect your **OpenRouter API** credential
   - Select the model you want to use in the UI if required
   - Leave default options unless you need custom temperature or model parameters

7. **Add a LangChain LLM Chain node**
   - Name it **Draft AI Reply**
   - Type: **LLM Chain**
   - Connect the **true** output of **Filter Unwanted Emails** into this node
   - Connect **OpenRouter Model** to the LLM Chain’s **AI language model** input
   - Set prompt mode to **Define**
   - In the main input text, assemble the incoming email:
     - `From: {{ $('Receive Customer Emails').item.json.from }}`
     - `Subject: {{ $('Receive Customer Emails').item.json.subject }}`
     - blank line
     - `{{ $('Receive Customer Emails').item.json.text }}`
   - Add an instruction message such as:
     - You are a support agent for your company
     - Include company documentation
     - Tell the model to answer only from that documentation
     - Tell it not to invent facts
     - Tell it to escalate politely when unsure
     - Tell it to sign as `Support Team`
     - Tell it to output only the email body, not a subject line

8. **Add a Gmail node to mark the message as read**
   - Name it **Mark Email as Read**
   - Type: **Gmail**
   - Connect **Draft AI Reply → Mark Email as Read**
   - Operation: **Mark as Read**
   - Set the message ID with:
     - `{{ $('Receive Customer Emails').item.json.id }}`
   - Reuse the same **Gmail OAuth2** credential

9. **Add a Google Sheets node to log the draft**
   - Name it **Log to Google Sheets**
   - Type: **Google Sheets**
   - Connect **Mark Email as Read → Log to Google Sheets**
   - Operation: **Append**
   - Choose your spreadsheet and sheet tab
   - Set mapping mode to manual/defined columns
   - Map:
     - `Message ID` → `{{ $('Receive Customer Emails').item.json.id }}`
     - `From` → `{{ $('Receive Customer Emails').item.json.from.value[0].address }}`
     - `Subject` → `{{ $('Receive Customer Emails').item.json.subject }}`
     - `Body` → `{{ $('Receive Customer Emails').item.json.text }}`
     - `Reply` → `{{ $('Draft AI Reply').item.json.text }}`
   - Leave `Send` blank so a human can review and type `send`

---

## Build Block 2: Review and Send

10. **Add a Schedule Trigger node**
    - Name it **Check Pending Replies**
    - Type: **Schedule Trigger**
    - Configure it to run every **1 minute**

11. **Add a Google Sheets node to fetch approved rows**
    - Name it **Get Approved Replies**
    - Type: **Google Sheets**
    - Connect **Check Pending Replies → Get Approved Replies**
    - Select the same spreadsheet and sheet tab
    - Configure it to read/filter rows where:
      - column: `Send`
      - value: `send`

12. **Add an IF node for approval confirmation**
    - Name it **Is Approved to Send?**
    - Connect **Get Approved Replies → Is Approved to Send?**
    - Add one condition:
      - `{{$json.Send}}` **equals** `send`
    - Keep only the true branch connected

13. **Add a Gmail send node**
    - Name it **Send Reply to Customer**
    - Type: **Gmail**
    - Connect **Is Approved to Send? (true) → Send Reply to Customer**
    - Set recipient to:
      - `{{ $json.From }}`
    - Set subject to:
      - `Re: {{ $json.Subject }}`
    - Set body to:
      - `{{ $json.Reply.replace(/\n/g, '<br>') }}`
    - Disable appended attribution/footer if the node offers that option
    - Connect the same **Gmail OAuth2** credential

14. **Add a Google Sheets update node**
    - Name it **Update Status to Replied**
    - Type: **Google Sheets**
    - Connect **Send Reply to Customer → Update Status to Replied**
    - Operation: **Append or Update**
    - Choose the same spreadsheet and sheet tab
    - Set matching column to:
      - `Message ID`
    - Map:
      - `Message ID` → `{{ $('Is Approved to Send?').item.json['Message ID'] }}`
      - `Send` → `Replied`

---

## Add documentation notes

15. **Add a sticky note for the first block**
    - Name: **Section - Email Intake**
    - Content:
      - `① Email Intake & AI Draft`
      - `Receives incoming emails, filters noise, drafts AI replies, and logs everything to Google Sheets for review.`

16. **Add a sticky note for the second block**
    - Name: **Section - Review and Send**
    - Content:
      - `② Review & Send Approved Replies`
      - `Checks Google Sheets every minute for rows marked "send" and delivers the approved reply to the customer.`

17. **Optionally add an overview sticky note**
    - Include purpose, setup notes, and operational instructions for reviewers.

---

## Final validation and activation

18. **Test the intake block**
    - Send a real test email into the Gmail inbox.
    - Confirm it passes the filter.
    - Confirm the AI generates a draft.
    - Confirm the message is marked read.
    - Confirm a row appears in Google Sheets.

19. **Test the approval block**
    - In the sheet, type `send` in the `Send` column for the test row.
    - Wait for the next scheduled run.
    - Confirm the reply email is sent.
    - Confirm the `Send` value becomes `Replied`.

20. **Activate the workflow**
    - Ensure both trigger nodes are enabled by activating the whole workflow.

---

## Credential and setup expectations

### Gmail OAuth2
- Must allow:
  - reading incoming messages for trigger data,
  - marking messages as read,
  - sending outgoing messages.

### Google Sheets OAuth2
- Must allow:
  - appending rows,
  - reading filtered rows,
  - updating rows by matching `Message ID`.

### OpenRouter
- Must have:
  - valid API key,
  - access to a chat-capable model,
  - sufficient quota/rate limits.

---

## Important implementation constraints

1. **Use exact column names**
   - `Message ID`, `From`, `Subject`, `Body`, `Reply`, `Send`

2. **Keep `Message ID` unique**
   - This field is used to update rows later.

3. **Customize the filter**
   - Replace `user@example.com` with your own address.
   - Consider adding case-insensitive normalization if needed.

4. **Customize the AI prompt**
   - Replace “Your Company Name Here”.
   - Replace placeholder support documentation with real internal support guidance.

5. **Understand threading limitations**
   - The send step uses `Re: {{Subject}}`, but it does not explicitly attach Gmail thread metadata.
   - If strict conversation threading is required, additional Gmail configuration may be needed.

6. **Prevent accidental duplicates**
   - If sending succeeds but sheet update fails, a later scheduled run may send the reply again.
   - A stronger design would update the row before sending or add a temporary “Sending” state.

7. **Review output formatting**
   - The workflow converts line breaks to `<br>`.
   - Verify Gmail renders this as intended in your n8n version.

8. **No sub-workflows are used**
   - There are no Execute Workflow or external sub-workflow dependencies in this JSON.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI customer support email agent with human approval loop | Overall workflow concept |
| Incoming emails are monitored every minute in Gmail | Intake block behavior |
| AI drafts are reviewed in Google Sheets before sending | Human-in-the-loop control |
| Type `send` in the `Send` column to approve a reply | Review process |
| Company documentation is embedded directly in the AI prompt rather than using a vector database or RAG | Knowledge source design |
| Suggested practical prompt size: around 10 pages / 3,000 words for broad model compatibility; larger context models can handle more | Prompt sizing guidance |
| Setup note: connect Gmail OAuth2 to trigger, mark-as-read, and send nodes | Credential requirement |
| Setup note: connect Google Sheets OAuth2 to all sheet nodes and create columns `Message ID`, `From`, `Subject`, `Body`, `Reply`, `Send` | Credential and schema requirement |
| Setup note: add OpenRouter API credentials to the OpenRouter Model node | Credential requirement |
| Setup note: replace placeholder business name and support documentation in the Draft AI Reply node | Prompt customization |
| Setup note: replace the excluded sender `user@example.com` with your own address in the filter node | Filter customization |