Triage Gmail inbox, draft replies, and alert urgent emails with Claude and Slack

https://n8nworkflows.xyz/workflows/triage-gmail-inbox--draft-replies--and-alert-urgent-emails-with-claude-and-slack-14852


# Triage Gmail inbox, draft replies, and alert urgent emails with Claude and Slack

Now I have a complete picture. Let me write the comprehensive document.### 1. Workflow Overview

This workflow is an **AI-powered Gmail inbox triage agent** that runs on a 15-minute polling schedule. It reads every new unread email, passes it to Anthropic's Claude Sonnet for intelligent classification into one of five categories (Urgent, Needs Reply, FYI Only, Automated, Spam), then takes automated action per category: Slack alerts for urgent items, draft replies for items needing a response, labeling and archiving for informational and automated mail, trashing for spam, and full audit logging to Google Sheets.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Input Reception & Data Preparation | Poll Gmail for unread emails every 15 minutes and extract sender, subject, and a 2 000-character body snippet. |
| 2 | AI Classification | Send the extracted email data to Claude Sonnet via a LangChain LLM chain, receive a structured JSON classification, and parse/validate it. |
| 3 | Category Routing | A Switch node dispatches each email to the correct action branch based on the parsed category. |
| 4 | Urgent Branch | Send a Slack alert, apply the AI-Urgent Gmail label. |
| 5 | Needs Reply Branch | Create a Gmail draft with Claude's suggested reply, apply the AI-Needs-Reply label. |
| 6 | FYI Only Branch | Mark the email as read and apply the AI-FYI label. |
| 7 | Automated Branch | Apply the AI-Automated label and archive the email. |
| 8 | Spam Branch | Move the email to trash. |
| 9 | Central Logging | Append a row to Google Sheets with the email's category, sender, subject, summary, reasoning, and draft status. All branches converge here. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Input Reception & Data Preparation

**Overview:** This block polls a Gmail inbox on a fixed schedule for unread messages and normalises each email into a compact data envelope (ID, sender, subject, body snippet, date, thread ID) suitable for downstream AI consumption while keeping token usage low.

**Nodes Involved:** Check for New Emails, Extract Email Data

---

**Node: Check for New Emails**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmailTrigger` (v1.2) |
| Role | Scheduled Gmail poller; the workflow's sole entry point. |
| Configuration | Filter: `readStatus = unread`. Polling: every 15 minutes (`everyX` mode, unit `minutes`, value `15`). |
| Key expressions/variables | None — this node outputs the raw Gmail message objects from the Gmail API. |
| Input | None (trigger node). |
| Output | Connects to **Extract Email Data**. Outputs one item per unread email with Gmail API fields (`id`, `threadId`, `from`, `subject`, `text`, `snippet`, `date`, etc.). |
| Version requirements | Gmail Trigger v1.2 requires a connected Gmail OAuth2 credential. |
| Edge cases / failures | — OAuth token expiration or revocation stops all polling. <br>— Gmail API rate limits (250 quota units per user per second) may cause failures on very large inboxes. <br>— If no unread emails exist, the workflow simply completes with zero items — downstream nodes receive nothing. <br>— Webhook ID is set; ensure the polling trigger is not accidentally switched to a webhook mode. |
| Sub-workflow | None |

---

**Node: Extract Email Data**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Data reshaping — extracts and truncates email fields into a clean envelope. |
| Configuration | Six assignments: `email_id` ← `$json.id`, `sender` ← `$json.from`, `subject` ← `$json.subject`, `body_snippet` ← `($json.text ∥ $json.snippet ∥ '').slice(0, 2000)`, `received_at` ← `$json.date`, `thread_id` ← `$json.threadId`. |
| Key expressions/variables | `$json.text`, `$json.snippet` — fallback chain ensures a snippet exists even when plain-text body is absent. The `.slice(0, 2000)` cap prevents excessive token usage in the AI step. |
| Input | **Check for New Emails**. |
| Output | Connects to **Classify Email Intent**. Outputs items with fields: `email_id`, `sender`, `subject`, `body_snippet`, `received_at`, `thread_id`. |
| Version requirements | Set node v3.4. |
| Edge cases / failures | — If both `text` and `snippet` are null/undefined, `body_snippet` resolves to an empty string — Claude will classify based on sender/subject alone, which may reduce accuracy. <br>— Multi-part MIME emails where `text` is empty but HTML body is present: only `snippet` would capture a small portion. |
| Sub-workflow | None |

---

#### Block 2 — AI Classification

**Overview:** This block sends each prepared email to Claude Sonnet via a LangChain LLM chain with a carefully crafted system prompt that forces structured JSON output. A Code node then extracts, validates, and enriches the JSON response, merging it back with the original email metadata.

**Nodes Involved:** Classify Email Intent, Claude Sonnet, Parse Classification

---

**Node: Classify Email Intent**

| Attribute | Detail |
|-----------|--------|
| Type | `@n8n/n8n-nodes-langchain.chainLlm` (v1.4) |
| Role | LangChain LLM chain — combines a system prompt with the Claude Sonnet model to classify each email. |
| Configuration | `promptType = define` (inline prompt). The prompt text is a detailed instruction set that tells the AI to act as an expert email triage assistant, read the email, and return **only** valid JSON with keys `category`, `summary`, `suggested_reply`, `reasoning`. Category must be one of: `URGENT`, `NEEDS_REPLY`, `FYI_ONLY`, `AUTOMATED`, `SPAM`. The prompt interpolates `{{ $json.sender }}`, `{{ $json.subject }}`, `{{ $json.body_snippet }}` from the incoming item. |
| Key expressions/variables | `{{ $json.sender }}`, `{{ $json.subject }}`, `{{ $json.body_snippet }}` — injected into the user prompt. |
| Input | **Extract Email Data** (main input). **Claude Sonnet** (ai_languageModel sub-node input). |
| Output | Connects to **Parse Classification**. The chain outputs `{ text: "<Claude's JSON response>" }`. |
| Version requirements | LangChain LLM Chain v1.4 requires the `@n8n/n8n-nodes-langchain` community package. |
| Edge cases / failures | — Claude may return markdown-fenced JSON (e.g. ```json … ```) or extra commentary despite the instruction — the downstream Parse Classification node handles this. <br>— If the Anthropic API key is invalid or quota exceeded, the chain fails with an authentication/rate-limit error. <br>— Extremely long emails (beyond 2000 chars) are truncated — classification may miss context from the lower portion of the email. |
| Sub-workflow | None (but it internally orchestrates the Claude Sonnet model sub-node). |

---

**Node: Claude Sonnet**

| Attribute | Detail |
|-----------|--------|
| Type | `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1.3) |
| Role | LLM model sub-node — provides the Anthropic Claude Sonnet 4.5 model to the parent chain. |
| Configuration | Model: `claude-sonnet-4-5`. Temperature: `0` (deterministic classification). |
| Key expressions/variables | None. |
| Input | Connected as `ai_languageModel` to **Classify Email Intent**. |
| Output | Returns model responses to the parent chain. |
| Version requirements | Requires an Anthropic credential (API key from `console.anthropic.com`). |
| Edge cases / failures | — API key must have sufficient credits/quota. <br>— `temperature: 0` ensures deterministic output but does not guarantee identical results across runs. <br>— If the model name is deprecated in the future, it must be updated here. |
| Sub-workflow | None |

---

**Node: Parse Classification**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.code` (v2) |
| Role | Post-processing — extracts the JSON object from Claude's text response, validates the category, and merges classification data with original email metadata. |
| Configuration | Inline JavaScript code that: (1) reads `$input.first().json.text` (the raw Claude output), (2) extracts the first `{…}` block via regex, (3) parses it as JSON with a fallback default (`FYI_ONLY`), (4) validates the category against the allowed list, (5) retrieves the original email data from the `Extract Email Data` node via `$('Extract Email Data').first().json`, (6) outputs a merged object. |
| Key expressions/variables | `$input.first().json.text` — Claude's raw text. `$('Extract Email Data').first().json` — reference to the earlier Set node's output (n8n's node reference syntax). `validCategories = ['URGENT','NEEDS_REPLY','FYI_ONLY','AUTOMATED','SPAM']`. |
| Input | **Classify Email Intent**. |
| Output | Connects to **Route by Category**. Outputs a single item with: `email_id`, `thread_id`, `sender`, `subject`, `body_snippet`, `received_at`, `category`, `summary`, `suggested_reply`, `reasoning`, `timestamp`. |
| Version requirements | Code node v2. |
| Edge cases / failures | — If Claude returns no JSON-like structure, the regex match fails and the default `FYI_ONLY` category is used with `summary = 'Could not parse classification'` and `reasoning = 'Parse error'`. <br>— If the parsed JSON contains an invalid category, it falls back to `FYI_ONLY`. <br>— The `$('Extract Email Data')` reference relies on node name — renaming the Set node breaks this reference. <br>— If `suggested_reply` is missing from Claude's output, it defaults to empty string. |
| Sub-workflow | None |

---

#### Block 3 — Category Routing

**Overview:** A Switch node inspects the `category` field from the parsed classification and routes each email item to one of five output branches, each corresponding to a distinct action pipeline.

**Nodes Involved:** Route by Category

---

**Node: Route by Category**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.switch` (v3.2) |
| Role | Conditional router — dispatches items to category-specific branches. |
| Configuration | Five rules, each using a string-equals condition on `={{ $json.category }}`: <br>• Output `Urgent` → `URGENT` <br>• Output `Needs Reply` → `NEEDS_REPLY` <br>• Output `FYI Only` → `FYI_ONLY` <br>• Output `Automated` → `AUTOMATED` <br>• Output `Spam` → `SPAM` <br>All outputs have `renameOutput = true`. Combinator is `and` for each rule (single condition per rule). |
| Key expressions/variables | `={{ $json.category }}` — the classification string from Parse Classification. |
| Input | **Parse Classification**. |
| Output | Five outputs: <br>0 → **Notify Urgent Email** <br>1 → **Save Draft Reply** <br>2 → **Label as FYI and Mark Read** <br>3 → **Label and Archive Automated** <br>4 → **Move Spam to Trash** |
| Version requirements | Switch v3.2. |
| Edge cases / failures | — If an item's `category` does not match any rule (should not happen given the Parse Classification validation), it is silently dropped — no fallback/unmatched output is configured. <br>— The `renameOutput = true` setting renames output tabs in the n8n UI for clarity but has no runtime effect. |
| Sub-workflow | None |

---

#### Block 4 — Urgent Branch

**Overview:** When an email is classified as Urgent, this branch immediately sends a Slack notification with the key details, then applies the AI-Urgent Gmail label to the message so the user can visually identify it in their inbox.

**Nodes Involved:** Notify Urgent Email, Label as Urgent

---

**Node: Notify Urgent Email**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.slack` (v2.2) |
| Role | Sends an alert message to a Slack channel for urgent emails. |
| Configuration | Channel selection: `select = channel`, `channelId` set by name `#urgent-emails`. Message text template: <br>`URGENT EMAIL\n\nFrom: {{ $json.sender }}\nSubject: {{ $json.subject }}\nSummary: {{ $json.summary }}\nReceived: {{ $json.received_at }}`. |
| Key expressions/variables | `{{ $json.sender }}`, `{{ $json.subject }}`, `{{ $json.summary }}`, `{{ $json.received_at }}`. |
| Input | **Route by Category** (output 0: Urgent). |
| Output | Connects to **Label as Urgent**. |
| Version requirements | Slack node v2.2 requires a Slack OAuth2 credential. |
| Edge cases / failures | — If the `#urgent-emails` channel does not exist or the bot is not a member, the message delivery fails. <br>— Slack API rate limits may trigger on burst urgent-classification events. <br>— If Slack is not used, this node should be disabled (right-click → Disable). <br>— The webhookId on this node is set but not used in polling mode. |
| Sub-workflow | None |

---

**Node: Label as Urgent**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Applies the AI-Urgent Gmail label to the urgent email. |
| Configuration | Operation: `addLabels`. `labelIds`: `["YOUR_AI_URGENT_LABEL_ID"]` (placeholder — must be replaced with the actual Gmail label ID). `messageId`: `={{ $json.email_id }}`. |
| Key expressions/variables | `={{ $json.email_id }}` — references the email ID from the parsed classification output. |
| Input | **Notify Urgent Email**. |
| Output | Connects to **Log to Sheets**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — The placeholder label ID `YOUR_AI_URGENT_LABEL_ID` must be replaced with a real Gmail label ID; otherwise the API call will fail with a 404 or invalid argument error. <br>— If the label does not exist in Gmail, the call fails. <br>— Gmail API label operations are idempotent — adding a label that already exists on the message does not cause an error. |
| Sub-workflow | None |

---

#### Block 5 — Needs Reply Branch

**Overview:** For emails that require a response, this branch creates a Gmail draft containing Claude's suggested reply (for human review before sending) and then applies the AI-Needs-Reply label.

**Nodes Involved:** Save Draft Reply, Label as Needs Reply

---

**Node: Save Draft Reply**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Creates a Gmail draft reply with Claude's suggested text, so the user can review and send it manually. |
| Configuration | Operation: `createDraft`. **Note:** The node configuration only specifies the operation — additional parameters (`to`, `subject`, `message` / `body`) are not set in the workflow JSON and must be manually configured. Expected expressions: `to` → `{{ $json.sender }}`, `subject` → `Re: {{ $json.subject }}`, `message` → `{{ $json.suggested_reply }}` (or the HTML body equivalent). |
| Key expressions/variables | Expected: `{{ $json.sender }}`, `{{ $json.subject }}`, `{{ $json.suggested_reply }}` — but these are not currently configured and must be added manually. |
| Input | **Route by Category** (output 1: Needs Reply). |
| Output | Connects to **Label as Needs Reply**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — **Missing parameters**: Without `to`, `subject`, and `body` fields, the `createDraft` operation will fail or create an empty draft. This is a critical configuration gap that must be resolved during setup. <br>— If `suggested_reply` is empty (e.g., Claude returned an empty string for a non-reply category), the draft will have no body text. <br>— Draft creation is subject to Gmail API quota limits. |
| Sub-workflow | None |

---

**Node: Label as Needs Reply**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Applies the AI-Needs-Reply Gmail label to the email. |
| Configuration | Operation: `addLabels`. `labelIds`: `["YOUR_AI_NEEDS_REPLY_LABEL_ID"]` (placeholder). `messageId`: `={{ $json.email_id }}`. |
| Key expressions/variables | `={{ $json.email_id }}`. |
| Input | **Save Draft Reply**. |
| Output | Connects to **Log to Sheets**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — Placeholder label ID must be replaced. <br>— Same idempotency and error conditions as Label as Urgent. |
| Sub-workflow | None |

---

#### Block 6 — FYI Only Branch

**Overview:** Emails that are purely informational are labeled as FYI and marked as read, keeping the inbox clean without requiring any action from the user.

**Nodes Involved:** Label as FYI and Mark Read

---

**Node: Label as FYI and Mark Read**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Marks the email as read. |
| Configuration | Operation: `markAsRead`. `messageId`: `={{ $json.email_id }}`. **Note:** The node name implies labeling as FYI, but the configuration only performs `markAsRead`. The AI-FYI label is **not** applied by this node. To fully implement the intended behavior, an additional `addLabels` operation with the AI-FYI label ID must be added, or a separate Gmail node must be inserted. |
| Key expressions/variables | `={{ $json.email_id }}`. |
| Input | **Route by Category** (output 2: FYI Only). |
| Output | Connects to **Log to Sheets**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — **Missing label application**: The FYI label is not applied despite the node name. This is a functional gap. <br>— `markAsRead` is idempotent. <br>— If the email ID is invalid, the operation fails. |
| Sub-workflow | None |

---

#### Block 7 — Automated Branch

**Overview:** System-generated emails (newsletters, notifications, receipts) are labeled and archived so they never clutter the inbox.

**Nodes Involved:** Label and Archive Automated

---

**Node: Label and Archive Automated**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Applies the AI-Automated label and should archive the email. |
| Configuration | Operation: `addLabels`. `labelIds`: `["YOUR_AI_AUTOMATED_LABEL_ID"]` (placeholder). `messageId`: `={{ $json.email_id }}`. **Note:** This node only adds a label; it does **not** remove the `INBOX` label, which is the Gmail API mechanism for archiving. To truly archive, an additional `removeLabels` operation with `INBOX` must be performed, either by changing this node's operation or adding a second Gmail node. |
| Key expressions/variables | `={{ $json.email_id }}`. |
| Input | **Route by Category** (output 3: Automated). |
| Output | Connects to **Log to Sheets**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — **Missing archive action**: The email is labeled but stays in the inbox. This is a functional gap. <br>— Placeholder label ID must be replaced. <br>— Same idempotency and error conditions as other label nodes. |
| Sub-workflow | None |

---

#### Block 8 — Spam Branch

**Overview:** Spam emails are immediately moved to trash.

**Nodes Involved:** Move Spam to Trash

---

**Node: Move Spam to Trash**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.1) |
| Role | Deletes (trashes) the spam email. |
| Configuration | Operation: `delete`. `messageId`: `={{ $json.email_id }}`. |
| Key expressions/variables | `={{ $json.email_id }}`. |
| Input | **Route by Category** (output 4: Spam). |
| Output | Connects to **Log to Sheets**. |
| Version requirements | Gmail node v2.1. |
| Edge cases / failures | — The `delete` operation in Gmail node v2.1 moves the message to Trash (not permanent delete). Messages in Trash are automatically purged after 30 days. <br>— If the email is already in Trash, this operation may still succeed (idempotent). <br>— No spam label is applied before trashing; consider adding `addLabels` with the Gmail Spam label ID if desired for tracking. <br>— False-positive spam classification could result in legitimate emails being trashed — review the audit log in Sheets regularly. |
| Sub-workflow | None |

---

#### Block 9 — Central Logging

**Overview:** Every processed email, regardless of category, is appended as a row in a Google Sheet, creating a persistent audit trail with the classification, summary, reasoning, and draft status.

**Nodes Involved:** Log to Sheets

---

**Node: Log to Sheets**

| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Role | Appends a row to a Google Sheet for every email processed, serving as the audit log. |
| Configuration | Operation: `append`. Sheet name: `Email Log` (by name). Document ID: `YOUR_GOOGLE_SHEET_ID` (placeholder — must be replaced). Column mapping mode: `defineBelow` with seven columns: <br>• `Sender` ← `{{ $json.sender }}` <br>• `Subject` ← `{{ $json.subject }}` <br>• `Summary` ← `{{ $json.summary }}` <br>• `Category` ← `{{ $json.category }}` <br>• `Reasoning` ← `{{ $json.reasoning }}` <br>• `Timestamp` ← `{{ $json.timestamp }}` <br>• `Draft Saved` ← `{{ $json.category === 'NEEDS_REPLY' ? 'Yes' : 'No' }}` |
| Key expressions/variables | `{{ $json.sender }}`, `{{ $json.subject }}`, `{{ $json.summary }}`, `{{ $json.category }}`, `{{ $json.reasoning }}`, `{{ $json.timestamp }}`, conditional `Draft Saved` expression. |
| Input | **Label as Urgent**, **Label as Needs Reply**, **Label as FYI and Mark Read**, **Label and Archive Automated**, **Move Spam to Trash** (all five branches converge here). |
| Output | None (terminal node). |
| Version requirements | Google Sheets node v4.5. Requires a Google OAuth2 credential. |
| Edge cases / failures | — Placeholder `YOUR_GOOGLE_SHEET_ID` must be replaced with a real sheet ID. <br>— The sheet must have a tab named exactly `Email Log` with matching column headers, otherwise data may be appended to the wrong columns or the operation will fail. <br>— Google Sheets API has a 500 cells per second per project rate limit — extremely high email volumes may hit this. <br>— If multiple branches fire concurrently (batch processing), items arrive as separate executions but the append operation handles each independently. <br>— The `Draft Saved` column uses a ternary expression; if `category` is not exactly `NEEDS_REPLY` (e.g., casing difference), it will output `No`. |
| Sub-workflow | None |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|-----------------|------------|
| Check for New Emails | gmailTrigger (v1.2) | Scheduled Gmail poller; entry point | — | Extract Email Data | **Fetch and prepare new emails** — Polls Gmail every 15 minutes for unread messages. Extracts the sender, subject, and first 2000 characters of the body — enough for Claude to classify without burning tokens on long email chains. |
| Extract Email Data | set (v3.4) | Reshape and truncate email fields | Check for New Emails | Classify Email Intent | **Fetch and prepare new emails** — Polls Gmail every 15 minutes for unread messages. Extracts the sender, subject, and first 2000 characters of the body — enough for Claude to classify without burning tokens on long email chains. |
| Classify Email Intent | @n8n/n8n-nodes-langchain.chainLlm (v1.4) | LangChain LLM chain with system prompt for email classification | Extract Email Data (main), Claude Sonnet (ai_languageModel) | Parse Classification | **Classify with Claude AI** — Claude Sonnet reads each email and returns a structured JSON with the category, a one-line summary, a suggested reply if needed, and its reasoning. Edit the system prompt in this node to add your name, your role, and any VIP senders or keywords that should always route as Urgent. |
| Claude Sonnet | @n8n/n8n-nodes-langchain.lmChatAnthropic (v1.3) | Anthropic Claude Sonnet 4.5 model sub-node | Classify Email Intent (as ai_languageModel) | Classify Email Intent (returns response) | **Classify with Claude AI** — Claude Sonnet reads each email and returns a structured JSON with the category, a one-line summary, a suggested reply if needed, and its reasoning. Edit the system prompt in this node to add your name, your role, and any VIP senders or keywords that should always route as Urgent. |
| Parse Classification | code (v2) | Extract JSON from Claude response, validate category, merge with email metadata | Classify Email Intent | Route by Category | **Classify with Claude AI** — Claude Sonnet reads each email and returns a structured JSON with the category, a one-line summary, a suggested reply if needed, and its reasoning. Edit the system prompt in this node to add your name, your role, and any VIP senders or keywords that should always route as Urgent. |
| Route by Category | switch (v3.2) | Conditional router dispatching to five category branches | Parse Classification | Notify Urgent Email, Save Draft Reply, Label as FYI and Mark Read, Label and Archive Automated, Move Spam to Trash | **Route by category** — A Switch node sends each email to the correct branch: Urgent, Needs Reply, FYI Only, Automated, or Spam. |
| Notify Urgent Email | slack (v2.2) | Send Slack alert for urgent emails | Route by Category (output 0) | Label as Urgent | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Label as Urgent | gmail (v2.1) | Apply AI-Urgent Gmail label | Notify Urgent Email | Log to Sheets | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Save Draft Reply | gmail (v2.1) | Create a Gmail draft with Claude's suggested reply | Route by Category (output 1) | Label as Needs Reply | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Label as Needs Reply | gmail (v2.1) | Apply AI-Needs-Reply Gmail label | Save Draft Reply | Log to Sheets | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Label as FYI and Mark Read | gmail (v2.1) | Mark email as read (FYI label application is missing) | Route by Category (output 2) | Log to Sheets | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Label and Archive Automated | gmail (v2.1) | Apply AI-Automated label (archiving not implemented) | Route by Category (output 3) | Log to Sheets | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Move Spam to Trash | gmail (v2.1) | Trash the spam email | Route by Category (output 4) | Log to Sheets | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |
| Log to Sheets | googleSheets (v4.5) | Append audit row to Google Sheets | Label as Urgent, Label as Needs Reply, Label as FYI and Mark Read, Label and Archive Automated, Move Spam to Trash | — | **Take action on each category** — Urgent: Slack alert sent immediately. Needs Reply: Claude draft saved to Gmail drafts. FYI Only: labelled and marked read. Automated: labelled and archived. Spam: labelled and trashed. Every email is then logged to Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "AI Gmail Inbox Triage Agent: Auto-Prioritise, Draft Replies and Alert with Claude".

2. **Add Schedule Trigger — Check for New Emails:**
   - Node type: `Gmail Trigger`
   - Connect your Gmail OAuth2 credential
   - Filter: `readStatus = unread`
   - Polling schedule: mode `everyX`, unit `minutes`, value `15`

3. **Add Extract Email Data:**
   - Node type: `Set` (Edit Fields)
   - Mode: manual assignment
   - Add six assignments:
     - `email_id` (string) → `={{ $json.id }}`
     - `sender` (string) → `={{ $json.from }}`
     - `subject` (string) → `={{ $json.subject }}`
     - `body_snippet` (string) → `={{ ($json.text || $json.snippet || '').slice(0, 2000) }}`
     - `received_at` (string) → `={{ $json.date }}`
     - `thread_id` (string) → `={{ $json.threadId }}`
   - Connect: **Check for New Emails** → **Extract Email Data**

4. **Add Classify Email Intent:**
   - Node type: `LangChain LLM Chain` (from `@n8n/n8n-nodes-langchain`)
   - Prompt type: `define` (inline)
   - Paste the full system prompt (see below) into the text field
   - Connect: **Extract Email Data** → **Classify Email Intent** (main input)

   **Prompt text to paste:**
   ```
   You are an expert email triage assistant for a busy business owner.

   Read the email below and classify it. Return ONLY valid JSON -- no extra text, no markdown, no explanation.

   EMAIL:
   From: {{ $json.sender }}
   Subject: {{ $json.subject }}
   Body: {{ $json.body_snippet }}

   Classify into exactly ONE of these categories:
   - URGENT: Needs the owner's attention today. Client issues, deal decisions, legal matters, anything time-sensitive.
   - NEEDS_REPLY: Requires a response but is not urgent. Questions, requests, follow-ups.
   - FYI_ONLY: Informational. No action needed. Updates, confirmations, read receipts.
   - AUTOMATED: System-generated. Newsletters, notifications, receipts, order confirmations, marketing emails.
   - SPAM: Unsolicited, irrelevant, or junk.

   Return this exact JSON structure:
   {
     "category": "URGENT",
     "summary": "One sentence describing what this email is about",
     "suggested_reply": "A short professional reply if category is NEEDS_REPLY, otherwise leave empty string",
     "reasoning": "One sentence explaining why you chose this category"
   }
   ```
   **Customization:** Add your name, role, and any VIP senders/keywords to the top of the prompt.

5. **Add Claude Sonnet model sub-node:**
   - Node type: `Anthropic Chat Model` (from `@n8n/n8n-nodes-langchain`)
   - Connect a new Anthropic credential (API key from `console.anthropic.com`)
   - Model: `claude-sonnet-4-5`
   - Temperature: `0`
   - Connect: **Claude Sonnet** → **Classify Email Intent** (as `ai_languageModel` sub-node connection — drag from the model node's `ai_languageModel` output to the chain's `ai_languageModel` input)

6. **Add Parse Classification:**
   - Node type: `Code`
   - Language: JavaScript
   - Paste the following code:
     ```javascript
     const text = $input.first().json.text || '';
     const emailData = $('Extract Email Data').first().json;

     let parsed = {
       category: 'FYI_ONLY',
       summary: 'Could not parse classification',
       suggested_reply: '',
       reasoning: 'Parse error'
     };

     try {
       const match = text.match(/\{[\s\S]*\}/);
       if (match) parsed = JSON.parse(match[0]);
     } catch(e) {}

     const validCategories = ['URGENT','NEEDS_REPLY','FYI_ONLY','AUTOMATED','SPAM'];
     if (!validCategories.includes(parsed.category)) parsed.category = 'FYI_ONLY';

     return [{
       json: {
         email_id:        emailData.email_id,
         thread_id:       emailData.thread_id,
         sender:          emailData.sender,
         subject:         emailData.subject,
         body_snippet:    emailData.body_snippet,
         received_at:     emailData.received_at,
         category:        parsed.category,
         summary:         parsed.summary,
         suggested_reply: parsed.suggested_reply || '',
         reasoning:       parsed.reasoning,
         timestamp:       new Date().toISOString()
       }
     }];
     ```
   - Connect: **Classify Email Intent** → **Parse Classification**

7. **Add Route by Category:**
   - Node type: `Switch`
   - Add five routing rules (all string equals on `={{ $json.category }}`):
     - Rule 1: Output key `Urgent`, right value `URGENT`
     - Rule 2: Output key `Needs Reply`, right value `NEEDS_REPLY`
     - Rule 3: Output key `FYI Only`, right value `FYI_ONLY`
     - Rule 4: Output key `Automated`, right value `AUTOMATED`
     - Rule 5: Output key `Spam`, right value `SPAM`
   - Enable `Rename Output` on each rule
   - Connect: **Parse Classification** → **Route by Category**

8. **Add Notify Urgent Email:**
   - Node type: `Slack`
   - Connect your Slack OAuth2 credential
   - Select: `channel`
   - Channel: `#urgent-emails` (create this channel in Slack first and invite the bot)
   - Message text:
     ```
     URGENT EMAIL

     From: {{ $json.sender }}
     Subject: {{ $json.subject }}
     Summary: {{ $json.summary }}
     Received: {{ $json.received_at }}
     ```
   - Connect: **Route by Category** (output 0) → **Notify Urgent Email**
   - **Optional:** Right-click and Disable this node if you do not use Slack.

9. **Add Label as Urgent:**
   - Node type: `Gmail`
   - Connect your Gmail OAuth2 credential
   - Operation: `addLabels`
   - Message ID: `={{ $json.email_id }}`
   - Label IDs: `[YOUR_AI_URGENT_LABEL_ID]` — replace with the actual Gmail label ID for your `AI-Urgent` label
   - Connect: **Notify Urgent Email** → **Label as Urgent**

10. **Add Save Draft Reply:**
    - Node type: `Gmail`
    - Connect your Gmail OAuth2 credential
    - Operation: `createDraft`
    - **Required additional parameters** (not in the original JSON — must be added manually):
      - `to`: `={{ $json.sender }}`
      - `subject`: `=Re: {{ $json.subject }}`
      - `message` / `body`: `={{ $json.suggested_reply }}`
    - Connect: **Route by Category** (output 1) → **Save Draft Reply**

11. **Add Label as Needs Reply:**
    - Node type: `Gmail`
    - Connect your Gmail OAuth2 credential
    - Operation: `addLabels`
    - Message ID: `={{ $json.email_id }}`
    - Label IDs: `[YOUR_AI_NEEDS_REPLY_LABEL_ID]` — replace with the actual Gmail label ID for your `AI-Needs-Reply` label
    - Connect: **Save Draft Reply** → **Label as Needs Reply**

12. **Add Label as FYI and Mark Read:**
    - Node type: `Gmail`
    - Connect your Gmail OAuth2 credential
    - Operation: `markAsRead`
    - Message ID: `={{ $json.email_id }}`
    - **Recommended fix:** Add a second parameter or a second Gmail node to perform `addLabels` with your `AI-FYI` label ID before or after marking as read, since the current configuration does not apply the FYI label.
    - Connect: **Route by Category** (output 2) → **Label as FYI and Mark Read**

13. **Add Label and Archive Automated:**
    - Node type: `Gmail`
    - Connect your Gmail OAuth2 credential
    - Operation: `addLabels`
    - Message ID: `={{ $json.email_id }}`
    - Label IDs: `[YOUR_AI_AUTOMATED_LABEL_ID]` — replace with the actual Gmail label ID for your `AI-Automated` label
    - **Recommended fix:** Add a second Gmail node or change this to also perform `removeLabels` with `INBOX` to actually archive the email (removing it from the inbox view).
    - Connect: **Route by Category** (output 3) → **Label and Archive Automated**

14. **Add Move Spam to Trash:**
    - Node type: `Gmail`
    - Connect your Gmail OAuth2 credential
    - Operation: `delete`
    - Message ID: `={{ $json.email_id }}`
    - Connect: **Route by Category** (output 4) → **Move Spam to Trash**

15. **Add Log to Sheets:**
    - Node type: `Google Sheets`
    - Connect your Google OAuth2 credential
    - Operation: `append`
    - Document ID: replace `YOUR_GOOGLE_SHEET_ID` with your actual Google Sheet ID (found in the sheet URL)
    - Sheet name: `Email Log` (by name)
    - Column mapping: `Define below`
    - Columns:
      - `Sender` ← `={{ $json.sender }}`
      - `Subject` ← `={{ $json.subject }}`
      - `Summary` ← `={{ $json.summary }}`
      - `Category` ← `={{ $json.category }}`
      - `Reasoning` ← `={{ $json.reasoning }}`
      - `Timestamp` ← `={{ $json.timestamp }}`
      - `Draft Saved` ← `={{ $json.category === 'NEEDS_REPLY' ? 'Yes' : 'No' }}`
    - Connect all five action endpoints to this node:
      - **Label as Urgent** → **Log to Sheets**
      - **Label as Needs Reply** → **Log to Sheets**
      - **Label as FYI and Mark Read** → **Log to Sheets**
      - **Label and Archive Automated** → **Log to Sheets**
      - **Move Spam to Trash** → **Log to Sheets**

16. **Create Gmail labels:**
    - In Gmail, create four labels: `AI-Urgent`, `AI-Needs-Reply`, `AI-FYI`, `AI-Automated`
    - To get label IDs: use Gmail API (Settings → Labels → hover over label → ID visible in URL) or call the Gmail API `users.labels.list` endpoint
    - Replace all placeholder label IDs in nodes 9, 11, 12, 13

17. **Create Google Sheet:**
    - Create a new Google Sheet
    - Name one tab `Email Log`
    - Add header row: `Sender`, `Subject`, `Summary`, `Category`, `Reasoning`, `Timestamp`, `Draft Saved`
    - Copy the Sheet ID from the URL and paste it into the Log to Sheets node

18. **Activate the workflow.**
    - Click the Active toggle in n8n. The workflow will now poll every 15 minutes automatically.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| **Your inbox is the most expensive place you spend time.** This workflow automates triage so you only see what genuinely needs your eyes. | Main workflow sticky note |
| Change the polling interval in the Schedule Trigger from 15 minutes to any frequency you prefer. | Customization tip from main sticky note |
| Add a VIP sender list to the Claude prompt so emails from specific people always route as Urgent. | Customization tip from main sticky note |
| Extend the Needs Reply branch to auto-send low-stakes replies without saving a draft. | Customization tip from main sticky note |
| If you do not use Slack, right-click the Notify Urgent Email node and select Disable. | Setup step from main sticky note |
| Claude writes the draft reply — you approve before sending. No email is ever sent automatically without your review. | Safety note from main sticky note |
| Anthropic API key can be obtained from `https://console.anthropic.com` | Credential setup |
| **Critical configuration gaps in the workflow JSON:** (1) Save Draft Reply is missing `to`, `subject`, and `body` parameters — must be added manually. (2) Label as FYI and Mark Read does not apply the FYI label — only marks as read. (3) Label and Archive Automated does not archive (remove INBOX label) — only adds a label. These must be fixed during setup. | Technical analysis |
| The `Parse Classification` Code node references `$('Extract Email Data')` by node name — renaming the Extract Email Data node will break this reference silently. | Technical caveat |
| All five action branches converge on a single Log to Sheets node. This means the Sheets node may receive items with different field compositions depending on the branch; ensure all required fields are present in every branch's output (they are, since Parse Classification produces a uniform schema). | Technical note |
| Gmail label IDs are numeric strings (e.g., `Label_1234567890`) — not the human-readable label names. Use the Gmail API `users.labels.list` to retrieve them. | Setup clarification |
| The `delete` operation on the Gmail node moves messages to Trash (30-day auto-purge), not permanent deletion. | Gmail behavior note |
| False-positive spam classification will trash legitimate emails. Review the Sheets audit log regularly and adjust the Claude prompt to reduce misclassification. | Operational safety |
| The workflow uses `executionOrder: v1` in settings, which means nodes execute in the order they appear in the `nodes` array, not based on connection topology. This is the default for workflows created in n8n versions prior to the v2 execution order. | Technical note |
| `binaryMode: separate` in workflow settings means binary data (email attachments) is handled separately and not merged into the main JSON items. | Technical note |