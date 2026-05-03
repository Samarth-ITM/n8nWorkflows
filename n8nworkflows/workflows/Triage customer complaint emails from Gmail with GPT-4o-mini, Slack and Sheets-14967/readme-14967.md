Triage customer complaint emails from Gmail with GPT-4o-mini, Slack and Sheets

https://n8nworkflows.xyz/workflows/triage-customer-complaint-emails-from-gmail-with-gpt-4o-mini--slack-and-sheets-14967


# Triage customer complaint emails from Gmail with GPT-4o-mini, Slack and Sheets

The user has provided a workflow JSON and a description. They want me to analyze the complete n8n workflow and create a comprehensive, structured reference document in English. Let me analyze the workflow JSON thoroughly.

Let me parse the workflow JSON to understand all nodes and connections.

**Nodes in the workflow:**

1. "Overview" - stickyNote - Large overview note
2. "Section — Inbox Monitoring and Config" - stickyNote
3. "Section — Email Parsing and Complaint Detection" - stickyNote
4. "Section — AI Triage Analysis" - stickyNote
5. "Section — Data Assembly" - stickyNote
6. "Section — Complaint Log and Department Alert" - stickyNote
7. "1. Gmail — Inbox Monitor" - gmailTrigger (typeVersion 1.3)
8. "2. Set — Config Values" - set (typeVersion 3.4)
9. "3. Code — Extract Email Fields" - code (typeVersion 2)
10. "4. IF — Is This a Complaint?" - if (typeVersion 2.2)
11. "5. AI Agent — Triage Complaint" - @n8n/n8n-nodes-langchain.agent (typeVersion 1.7)
12. "6. OpenAI — GPT-4o-mini Model" - @n8n/n8n-nodes-langchain.lmChatOpenAi (typeVersion 1.2)
13. "7. Parser — Structured Triage Output" - @n8n/n8n-nodes-langchain.outputParserStructured (typeVersion 1.3)
14. "8. Code — Combine Triage Data" - code (typeVersion 2)
15. "9. Google Sheets — Log Complaint" - googleSheets (typeVersion 4.5)
16. "10. Slack — Send Department Alert" - slack (typeVersion 2.2)
17. "11. NoOp — Not a Complaint" - noOp (typeVersion 1)

**Connections:**
- Gmail → Set (Config Values)
- Set (Config Values) → Code (Extract Email Fields)
- Code (Extract Email Fields) → IF (Is This a Complaint?)
- IF (Complaint YES) → AI Agent (Triage Complaint)
- IF (Complaint NO) → NoOp (Not a Complaint)
- OpenAI Model → AI Agent (as ai_languageModel)
- Parser → AI Agent (as ai_outputParser)
- AI Agent → Code (Combine Triage Data)
- Code (Combine Triage Data) → Google Sheets (Log Complaint)
- Code (Combine Triage Data) → Slack (Send Department Alert)

Now let me organize this into logical blocks:

**Block 1: Input Reception and Configuration** (Inbox Monitoring and Config)
- Gmail Trigger
- Set Config Values

**Block 2: Email Parsing and Complaint Detection** (Email Parsing and Complaint Detection)
- Code — Extract Email Fields
- IF — Is This a Complaint?
- NoOp — Not a Complaint

**Block 3: AI Triage Analysis** (AI Triage Analysis)
- AI Agent — Triage Complaint
- OpenAI — GPT-4o-mini Model
- Parser — Structured Triage Output

**Block 4: Data Assembly** (Data Assembly)
- Code — Combine Triage Data

**Block 5: Output and Notification** (Complaint Log and Department Alert)
- Google Sheets — Log Complaint
- Slack — Send Department Alert

Now let me create the comprehensive document.

For sticky notes, I need to map them to the nodes they cover:

The sticky notes are:
1. "Overview" - large note covering the full workflow
2. "Section — Inbox Monitoring and Config" - covers Gmail Trigger and Set Config Values
3. "Section — Email Parsing and Complaint Detection" - covers Extract Email Fields, IF node, NoOp
4. "Section — AI Triage Analysis" - covers AI Agent, OpenAI Model, Parser
5. "Section — Data Assembly" - covers Combine Triage Data
6. "Section — Complaint Log and Department Alert" - covers Google Sheets and Slack

For the summary table, I need to assign sticky note content to each affected node.

Let me now create the full document.

For node details, I need to describe each node's configuration:

**1. Gmail — Inbox Monitor**
- Type: gmailTrigger
- Polls every minute
- simple: false (returns full message data)
- No filters applied
- Credential: Gmail OAuth2 required

**2. Set — Config Values**
- Type: set (v3.4)
- Assignments:
  - companyName: string (default "YOUR COMPANY NAME")
  - sheetId: string (default "YOUR_GOOGLE_SHEET_ID")
  - sheetName: string (default "Complaint Log")
  - slackBillingChannel: string (default "#billing-support")
  - slackTechChannel: string (default "#tech-support")
  - slackSupportChannel: string (default "#customer-support")
  - supportEmail: string (default "user@example.com")

**3. Code — Extract Email Fields**
- Type: code (v2)
- Extracts: senderEmail, senderName, subject, emailBody (truncated to 3000 chars), receivedAt, emailId
- Also passes through config values
- References the previous Set node to get config values

**4. IF — Is This a Complaint?**
- Type: if (v2.2)
- Single boolean condition checking subject and emailBody for 12 complaint keywords
- Case insensitive
- YES path → AI Agent
- NO path → NoOp

The keywords checked (from the expression):
- subject: complaint, issue, problem, not working, broken, refund, angry, disappointed
- emailBody: complaint, very unhappy, unacceptable, demand refund, worst

**5. AI Agent — Triage Complaint**
- Type: @n8n/n8n-nodes-langchain.agent (v1.7)
- promptType: define
- hasOutputParser: true
- Detailed prompt asking for JSON with 4 fields: urgencyScore, department, oneLineSummary, suggestedReply
- Connected to OpenAI model and Structured Output Parser

**6. OpenAI — GPT-4o-mini Model**
- Type: @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.2)
- model: gpt-4o-mini
- temperature: 0.3
- maxTokens: 600
- Connected to AI Agent as ai_languageModel

**7. Parser — Structured Triage Output**
- Type: @n8n/n8n-nodes-langchain.outputParserStructured (v1.3)
- schemaType: manual
- JSON schema with 4 required properties: urgencyScore (number), department (string), oneLineSummary (string), suggestedReply (string)
- Connected to AI Agent as ai_outputParser

**8. Code — Combine Triage Data**
- Type: code (v2)
- Merges AI output with email data from node 3
- Sets urgencyLabel based on score thresholds:
  - >= 8: CRITICAL
  - >= 6: High
  - >= 4: Medium
  - else: Low
- Routes Slack channel based on department
- Formats receivedDate
- Adds loggedAt timestamp

**9. Google Sheets — Log Complaint**
- Type: googleSheets (v4.5)
- operation: append
- sheetName: dynamic from $json.sheetName
- documentId: dynamic from $json.sheetId
- columns mapped to 12 fields
- cellFormat: USER_ENTERED

**10. Slack — Send Department Alert**
- Type: slack (v2.2)
- authentication: oAuth2
- mrkdwn: true
- channel dynamic from $json.slackChannel
- Formatted message with urgency label, score, department, sender info, subject, summary, suggested reply

**11. NoOp — Not a Complaint**
- Type: noOp (v1)
- No configuration, terminal node for non-complaint emails

Now let me write the comprehensive document.

For the summary table, sticky notes need to be mapped:

- Overview sticky note covers ALL nodes (it's a general overview)
- Section — Inbox Monitoring and Config covers nodes 1 and 2
- Section — Email Parsing and Complaint Detection covers nodes 3, 4, and 11
- Section — AI Triage Analysis covers nodes 5, 6, and 7
- Section — Data Assembly covers node 8
- Section — Complaint Log and Department Alert covers nodes 9 and 10

I'll include the sticky note content for each relevant node in the table.

Actually, looking at the sticky note content more carefully, each section sticky note has a brief description. I should include those in the relevant rows. The Overview sticky note is very long - I'll include it for all nodes.

Actually, let me re-read the instructions about sticky notes:

" The `Sticky Note` column must include any associated comment (no sticky note ID). If a sticky note visually covers multiple nodes, duplicate its content in each affected row."

So I need to duplicate the relevant sticky note content for each node it covers.

This would make the table very verbose with the Overview note. Let me think about this... The Overview is a large note that visually covers all nodes (position [-1120,-368] with width 540, height 1040). I should include it for all nodes. But that would make every row very long. Let me include a shortened version or just reference it.

Actually, I'll include the full content of each sticky note for each node it covers, as instructed. But I'll format it reasonably.

Let me reconsider - the Overview note is quite long. Let me include it for all nodes but perhaps in a more concise way. Actually, the instructions say "duplicate its content in each affected row." So I should include the full content. But that would make the table extremely long. Let me include a summary reference instead.

Wait, let me re-read: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row."

The overview note has very long content. Including it for all 11 data nodes would be impractical. I think I should include the section-specific sticky notes for each node, and note the Overview for all nodes but perhaps truncate or reference it.

Actually, looking at the positions:
- Overview: position [-1120,-368], width 540, height 1040 - this is a large note on the left side
- Section notes are positioned near their respective nodes

Let me include the section-specific sticky notes for each node, and for the Overview, I'll include it for all nodes but perhaps abbreviated.

Actually, let me just follow the instructions literally. I'll include the full sticky note content for each node it covers. For the Overview, since it covers all nodes, I'll include a reference to it.

Hmm, but the Overview content is very long. Let me include a truncated version like "Overview: Customer Complaint Triage — Gmail + GPT-4o-mini + Slack + Google Sheets..." for all nodes, and then include the section-specific notes in full.

Let me be practical. I'll include the section-specific sticky notes for each node, and note that the Overview sticky note applies to all nodes.

Actually, I think the best approach is to include the most relevant sticky note content for each node. The Overview is a general description that applies to the whole workflow, so I'll note it separately. The section-specific notes are more relevant.

Let me structure this as:

For each node, include the section-specific sticky note that covers it.

OK, let me now write the full document.

Actually, I realize I should also check the positions more carefully to determine which sticky notes cover which nodes:

Node positions:
- Gmail: [-512, 80]
- Set Config: [-288, 80]
- Code Extract: [-64, 80]
- IF: [160, 80]
- AI Agent: [368, 64]
- OpenAI: [368, 288]
- Parser: [512, 384]
- Code Combine: [816, 64]
- Google Sheets: [1152, -144]
- Slack: [1200, 208]
- NoOp: [160, 288]

Sticky note positions:
- Overview: [-1120, -368], width 540, height 1040 → covers x: -1120 to -580, y: -368 to 672
  - This doesn't seem to cover any nodes (nodes start at x=-512)
  - Actually wait, it might not visually cover the nodes. Let me check...
  - Gmail is at [-512, 80], which is outside the Overview note's x range (-1120 to -580)
  - Actually, the overview is probably just a documentation note, not visually covering any specific node

- Section Inbox Monitoring: [-560, -96], width 436, height 340 → covers x: -560 to -124, y: -96 to 244
  - Gmail [-512, 80] is within this range ✓
  - Set Config [-288, 80] is within this range ✓

- Section Email Parsing: [-112, -176], width 420, height 660 → covers x: -112 to 308, y: -176 to 484
  - Code Extract [-64, 80] is within this range ✓
  - IF [160, 80] is within this range ✓
  - NoOp [160, 288] is within this range ✓

- Section AI Triage: [336, -176], width 404, height 708 → covers x: 336 to 740, y: -176 to 532
  - AI Agent [368, 64] is within this range ✓
  - OpenAI [368, 288] is within this range ✓
  - Parser [512, 384] is within this range ✓

- Section Data Assembly: [752, -176], width not specified but height 420 → covers x: 752 to ?, y: -176 to 244
  - Wait, the parameters show height: 420 but no width. Default sticky note width is probably around 240.
  - Actually looking at it more carefully, it has "height": 420 but I don't see a width parameter. The default is probably 240.
  - So covers x: 752 to ~992, y: -176 to 244
  - Code Combine [816, 64] is within this range ✓

- Section Complaint Log: [1040, -320], width 408, height 804 → covers x: 1040 to 1448, y: -320 to 484
  - Google Sheets [1152, -144] is within this range ✓
  - Slack [1200, 208] is within this range ✓

Great, so my grouping is correct. Now let me write the document.

For the Overview sticky note, it doesn't visually cover any specific node (it's positioned far to the left), so I won't include it in the sticky note column for individual nodes.

Let me finalize the document now.

I need to also document the connections between nodes clearly, including the special connections for AI Agent (languageModel and outputParser).

Let me also note the important details about each node:

For the AI Agent, the prompt is quite detailed and includes specific rules. Let me capture that.

For the IF node, the condition is a single boolean expression checking 12 keywords.

For the Set node, the config values are important placeholders that need to be replaced.

OK, I think I have everything. Let me write the comprehensive document now.

One more thing - I need to make sure I document edge cases and potential failures for each node.

Let me think about edge cases:

1. Gmail Trigger:
   - OAuth2 token expiration
   - Rate limiting (polling every minute)
   - Empty inbox
   - Large attachments

2. Set Config:
   - Missing/incorrect values
   - Sheet ID format issues

3. Code Extract Email Fields:
   - Missing from field
   - HTML-only emails (no text body)
   - Very short emails
   - Unicode in sender names

4. IF Complaint Check:
   - False positives from partial word matches
   - Case sensitivity (handled by toLowerCase)
   - Non-English emails

5. AI Agent:
   - API errors
   - Timeout
   - Rate limiting
   - Malformed output (partially handled by parser)

6. OpenAI Model:
   - Token limit exceeded
   - API key issues
   - Model availability

7. Parser:
   - Schema validation failures
   - Missing required fields

8. Code Combine Triage Data:
   - Missing AI output fields (fallbacks provided)
   - Unknown department values (defaults to General channel)

9. Google Sheets:
   - Sheet not found
   - Sheet tab name mismatch
   - Permission issues
   - Column header mismatch

10. Slack:
    - Channel not found
    - Bot not invited to channel
    - Message formatting issues
    - Rate limiting

11. NoOp:
    - No edge cases (terminal node)

Now let me write the full document.

For the reproducing section, I need step-by-step instructions to rebuild from scratch.

Let me structure it:

1. Create the Gmail Trigger node
2. Create the Set Config Values node
3. Connect Gmail → Set
4. Create the Code Extract Email Fields node
5. Connect Set → Code Extract
6. Create the IF node
7. Connect Code Extract → IF
8. Create the NoOp node
9. Connect IF (false) → NoOp
10. Create the AI Agent node
11. Connect IF (true) → AI Agent
12. Create the OpenAI Model node
13. Connect OpenAI → AI Agent (languageModel)
14. Create the Parser node
15. Connect Parser → AI Agent (outputParser)
16. Create the Code Combine Triage Data node
17. Connect AI Agent → Code Combine
18. Create the Google Sheets node
19. Create the Slack node
20. Connect Code Combine → Google Sheets
21. Connect Code Combine → Slack

This is the correct order based on dependencies.

Now let me write the full document.

I also need to document the AI Agent prompt in detail.

The AI Agent prompt is:
```
You are a customer support triage specialist at {{ $json.companyName }}.

You must analyze this customer complaint email and return a JSON object with exactly 4 fields.

CUSTOMER EMAIL:
From: {{ $json.senderName }} ({{ $json.senderEmail }})
Subject: {{ $json.subject }}
Body: {{ $json.emailBody }}

You must return ONLY a valid JSON object with these 4 fields:

urgencyScore — a number from 1 to 10 based on how urgent this complaint is.
1 = minor inconvenience, 5 = moderate issue, 8 = very upset customer, 10 = threatening legal action or churn

department — which team should handle this complaint. Choose exactly one of these:
Billing (payment issues, refund requests, invoice problems, overcharge)
Technical (product not working, bugs, errors, login issues, integration problems)
General (shipping, delivery, service quality, general dissatisfaction)

oneLineSummary — one plain sentence (under 20 words) summarizing what the customer is complaining about.

suggestedReply — a professional, empathetic reply the support agent can send to this customer. Keep it under 80 words. Plain text only. Start with the customer name. End with your company name.

RULES:
- Return ONLY the JSON object. No extra text before or after.
- No markdown. No code blocks. No backticks.
- urgencyScore must be a number, not a string.
- department must be exactly: Billing OR Technical OR General
```

The Parser schema:
```json
{
  "type": "object",
  "properties": {
    "urgencyScore": {
      "type": "number",
      "description": "Urgency score from 1 to 10. 1 = minor, 10 = critical"
    },
    "department": {
      "type": "string",
      "description": "Which department: Billing OR Technical OR General"
    },
    "oneLineSummary": {
      "type": "string",
      "description": "One sentence summary of the complaint under 20 words"
    },
    "suggestedReply": {
      "type": "string",
      "description": "Professional empathetic reply under 80 words for the support agent to send"
    }
  },
  "required": ["urgencyScore", "department", "oneLineSummary", "suggestedReply"]
}
```

OK, I have all the information needed. Let me write the comprehensive document now.# Customer Complaint Triage — Gmail + GPT-4o-mini + Slack + Google Sheets

---

## 1. Workflow Overview

This workflow continuously monitors a Gmail inbox, detects customer complaint emails using keyword matching, routes each complaint through GPT‑4o‑mini for AI‑driven triage (urgency scoring, department classification, one‑line summary, and reply draft), then permanently logs every complaint to Google Sheets and posts a formatted alert to the appropriate Slack channel. Non‑complaint emails exit silently without triggering any logging, AI calls, or notifications.

**Logical Blocks:**

| Block | Purpose |
|---|---|
| **1.1 Input Reception & Configuration** | Polls Gmail every minute for new emails and stores all user‑configurable values (company name, Sheet ID, Slack channels, support email) in a single Set node for downstream reuse. |
| **1.2 Email Parsing & Complaint Detection** | Extracts sender, subject, and body from the raw Gmail payload; truncates the body to 3 000 characters; checks subject and body against 12 complaint keywords; non‑complaints exit via a NoOp terminal. |
| **1.3 AI Triage Analysis** | Sends the cleaned email data to an AI Agent powered by GPT‑4o‑mini; a Structured Output Parser enforces a four‑field JSON schema (urgency score, department, summary, suggested reply). |
| **1.4 Data Assembly** | Merges AI output with the original email data; converts the numeric urgency score into a human‑readable label (Low / Medium / High / CRITICAL); selects the correct Slack channel based on department; adds a logged‑at timestamp. |
| **1.5 Output — Logging & Alerting** | Appends a 12‑column row to a Google Sheets "Complaint Log" tab and posts a richly formatted Slack message to the department‑specific channel. Both actions run in parallel after data assembly. |

---

## 2. Block‑by‑Block Analysis

### Block 1.1 — Input Reception & Configuration

**Overview:** This block is the workflow's entry point. The Gmail Trigger node polls the configured inbox every 60 seconds and emits every new message. Immediately after, a Set node stores seven configuration values that all downstream nodes reference, ensuring a single point of maintenance.

**Nodes Involved:**
- 1. Gmail — Inbox Monitor
- 2. Set — Config Values

---

#### Node: 1. Gmail — Inbox Monitor

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.gmailTrigger` (v1.3) |
| **Functional Role** | Entry trigger; polls the connected Gmail inbox for new messages every minute and passes each message downstream. |
| **Configuration** | • `simple`: false (returns full message payload, not just headers) • `pollTimes`: everyMinute (60‑second interval) • No label or query filters applied |
| **Credential** | Gmail OAuth2 — must be authorised for read‑only inbox access. |
| **Input** | None (trigger node). |
| **Output** | One item per new email containing the full Gmail message object (headers, body, from, subject, date, id, etc.). |
| **Edge Cases / Failure Types** | • OAuth2 token expiration → re‑authorise the credential. • Gmail API rate limiting → n8n will retry automatically, but high‑volume inboxes may need increased poll interval. • Empty inbox → node emits no items, downstream pipeline is idle. • Attachments are passed through but not used by this workflow. |

---

#### Node: 2. Set — Config Values

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Functional Role** | Centralised configuration store; defines seven named variables consumed by every downstream node. |
| **Configuration** | Seven string assignments: |
| | • `companyName` → `"YOUR COMPANY NAME"` (replace with actual company name) |
| | • `sheetId` → `"YOUR_GOOGLE_SHEET_ID"` (the ID fragment from the Sheet URL) |
| | • `sheetName` → `"Complaint Log"` (must match the Google Sheets tab name exactly) |
| | • `slackBillingChannel` → `"#billing-support"` |
| | • `slackTechChannel` → `"#tech-support"` |
| | • `slackSupportChannel` → `"#customer-support"` |
| | • `supportEmail` → `"user@example.com"` (replace with real support address) |
| **Credential** | None. |
| **Input** | Output of Gmail Trigger (the email item is passed through unchanged; config values are added). |
| **Output** | Original email payload plus the seven config fields. |
| **Edge Cases / Failure Types** | • Leaving placeholder values (e.g. `YOUR_GOOGLE_SHEET_ID`) will cause downstream Google Sheets node to fail with "Spreadsheet not found". • Channel names without `#` prefix or with typos will cause Slack node to fail with "channel_not_found". • Mismatched `sheetName` casing will cause "Sheet not found" error. |

---

### Block 1.2 — Email Parsing & Complaint Detection

**Overview:** This block transforms raw Gmail payload fields into clean, workflow‑ready variables, truncates the email body to 3 000 characters, and applies a keyword‑based filter. Emails matching at least one of twelve complaint signals continue to AI triage; all others exit silently through a NoOp terminal.

**Nodes Involved:**
- 3. Code — Extract Email Fields
- 4. IF — Is This a Complaint?
- 11. NoOp — Not a Complaint

---

#### Node: 3. Code — Extract Email Fields

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Functional Role** | Parses the Gmail message object, extracts sender name/email, subject, and body, truncates the body, and merges the config values into a single flat object for downstream use. |
| **Configuration** | JavaScript code that: |
| | • Reads `from.text` (or `from`) and extracts `senderEmail` via regex and `senderName` by stripping the angle‑bracket portion. |
| | • Falls back to `"user@example.com"` and `"Unknown Sender"` when parsing fails. |
| | • Reads `subject`, `text`, `snippet`, or `body.text` for content, defaulting to `"No Subject"` / `"No email body found"`. |
| | • Truncates `emailBody` to 3 000 characters to limit token usage. |
| | • Reads `date` (or generates current timestamp) and `id` / `messageId` for the email ID. |
| | • Merges in all seven config values from node 2 for downstream convenience. |
| **Key Expressions** | `$('2. Set — Config Values').item.json` to pull config values; regex `[\\w.-]+@[\\w.-]+\\.[a-z]{2,}` for email extraction; `body.substring(0, 3000)` for truncation. |
| **Credential** | None. |
| **Input** | Output of node 2 (email + config). |
| **Output** | Flat JSON object with keys: `emailId`, `senderEmail`, `senderName`, `subject`, `emailBody`, `receivedAt`, plus all seven config fields (`companyName`, `sheetId`, `sheetName`, `slackBillingChannel`, `slackTechChannel`, `slackSupportChannel`, `supportEmail`). |
| **Edge Cases / Failure Types** | • HTML‑only emails with no plain‑text part may yield `"No email body found"` unless `snippet` is available. • Malformed `from` header (missing angle brackets) falls back to defaults. • Unicode sender names are preserved but may cause display issues in Slack if not properly escaped. |

---

#### Node: 4. IF — Is This a Complaint?

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.if` (v2.2) |
| **Functional Role** | Binary filter that directs complaint emails to AI triage (TRUE branch) and discards non‑complaints (FALSE branch). |
| **Configuration** | Single boolean condition using an expression that checks 12 complaint keywords (case‑insensitive) in `subject` and `emailBody`: |
| | **Subject keywords:** complaint, issue, problem, not working, broken, refund, angry, disappointed |
| | **Body keywords:** complaint, very unhappy, unacceptable, demand refund, worst |
| | Expression: `{{ $json.subject.toLowerCase().includes('complaint') \|\| $json.subject.toLowerCase().includes('issue') \|\| … \|\| $json.emailBody.toLowerCase().includes('worst') }}` |
| **Logic** | TRUE (any keyword match) → node 5 (AI Agent). FALSE (no match) → node 11 (NoOp). |
| **Credential** | None. |
| **Input** | Output of node 3. |
| **Output (TRUE)** | Same item passed to node 5. |
| **Output (FALSE)** | Same item passed to node 11. |
| **Edge Cases / Failure Types** | • Partial‑word matches may cause false positives (e.g. "issue" matching "tissue"). • Emails entirely in non‑English languages will almost always evaluate to FALSE. • Very long subjects/emails may cause the expression to exceed n8n's expression length limit (rare). • Adding more keywords requires editing the long inline expression. |

---

#### Node: 11. NoOp — Not a Complaint

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.noOp` (v1) |
| **Functional Role** | Terminal sink for non‑complaint emails; performs no action and ends the branch cleanly. |
| **Configuration** | None (default). |
| **Credential** | None. |
| **Input** | FALSE branch of node 4. |
| **Output** | None (workflow ends for this item). |
| **Edge Cases / Failure Types** | None — this node never fails. |

---

### Block 1.3 — AI Triage Analysis

**Overview:** This block is the intelligence core. An AI Agent powered by GPT‑4o‑mini receives the cleaned email data and returns a structured JSON object with four fields: urgency score (1–10), department (Billing / Technical / General), one‑line summary, and a suggested reply. A Structured Output Parser enforces the schema, preventing malformed responses from propagating.

**Nodes Involved:**
- 5. AI Agent — Triage Complaint
- 6. OpenAI — GPT‑4o‑mini Model
- 7. Parser — Structured Triage Output

---

#### Node: 5. AI Agent — Triage Complaint

| Property | Detail |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v1.7) |
| **Functional Role** | Orchestrates the AI call: sends the customer email data with a detailed system prompt and receives a JSON triage result, validated by the connected output parser. |
| **Configuration** | • `promptType`: define (prompt is explicitly set in the node) • `hasOutputParser`: true (connected to node 7) • `text`: Full prompt (see below) |
| **Prompt (full text)** | |
| | *System instruction:* "You are a customer support triage specialist at `{{ $json.companyName }}`." |
| | *Task:* Analyse the email and return **only** a JSON object with four fields: |
| | – `urgencyScore` (number 1–10; 1 = minor inconvenience, 5 = moderate, 8 = very upset, 10 = legal/churn threat) |
| | – `department` (exactly one of: `Billing`, `Technical`, `General`) |
| | – `oneLineSummary` (plain sentence, ≤ 20 words) |
| | – `suggestedReply` (professional, empathetic, ≤ 80 words, starts with customer name, ends with company name) |
| | *Rules:* Return only JSON, no markdown or backticks; `urgencyScore` must be a number; `department` must be exactly one of the three allowed values. |
| | *Dynamic fields injected:* `{{ $json.senderName }}`, `{{ $json.senderEmail }}`, `{{ $json.subject }}`, `{{ $json.emailBody }}`, `{{ $json.companyName }}` |
| **Connected Sub‑nodes** | • `ai_languageModel` ← node 6 (OpenAI) • `ai_outputParser` ← node 7 (Parser) |
| **Credential** | None directly (credential is on node 6). |
| **Input** | TRUE branch of node 4 (complaint email data). |
| **Output** | JSON object with `output` key containing the four parsed fields (`urgencyScore`, `department`, `oneLineSummary`, `suggestedReply`). |
| **Edge Cases / Failure Types** | • OpenAI API downtime → node fails, workflow execution stops for that item. • Token limit exceeded (body + prompt > model context) → truncation already handled by node 3 (3 000 char cap). • Model returns non‑JSON text → Parser node catches and re‑prompts or fails. • Rate limiting on OpenAI side → n8n retries with exponential backoff. |

---

#### Node: 6. OpenAI — GPT‑4o‑mini Model

| Property | Detail |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) |
| **Functional Role** | Provides the language model backend for the AI Agent; configured for deterministic, cost‑efficient responses. |
| **Configuration** | • `model`: gpt‑4o‑mini (selected from list) • `temperature`: 0.3 (low, for consistent structured output) • `maxTokens`: 600 (caps response length and cost) |
| **Credential** | OpenAI API key — must have access to the `gpt-4o-mini` model. |
| **Input** | Connected to node 5 via `ai_languageModel` input. |
| **Output** | Model response consumed by node 5. |
| **Edge Cases / Failure Types** | • Invalid or expired API key → 401 authentication error. • Insufficient credits → 429/402 error. • Model not available on the key's plan → 403 error. • 600‑token cap may truncate very long replies (unlikely given prompt constraints). |

---

#### Node: 7. Parser — Structured Triage Output

| Property | Detail |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) |
| **Functional Role** | Enforces the exact JSON schema the AI Agent must return; validates types and required fields before passing data downstream. |
| **Configuration** | • `schemaType`: manual • `inputSchema`: JSON schema with four required properties — `urgencyScore` (number, 1–10), `department` (string, one of Billing/Technical/General), `oneLineSummary` (string), `suggestedReply` (string) |
| **Credential** | None. |
| **Input** | Connected to node 5 via `ai_outputParser` input. |
| **Output** | Parsed and validated JSON object consumed by node 5. |
| **Edge Cases / Failure Types** | • AI returns `urgencyScore` as a string (e.g. `"7"`) → parser will coerce or reject. • AI returns a department value outside the three allowed options → parser validation fails, workflow errors for that item. • Missing required field → parser fails, execution stops. |

---

### Block 1.4 — Data Assembly

**Overview:** This block merges the AI triage output with the original email metadata, converts the numeric urgency score into a human‑readable label, selects the correct Slack channel based on department, and adds a timestamp for the log.

**Nodes Involved:**
- 8. Code — Combine Triage Data

---

#### Node: 8. Code — Combine Triage Data

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Functional Role** | Merges AI output with the original email data from node 3; adds derived fields (urgency label, Slack channel, formatted dates) for downstream logging and alerting. |
| **Configuration** | JavaScript code that: |
| | • Reads AI output from `$input.first().json.output` and email data from `$('3. Code — Extract Email Fields').item.json`. |
| | • Applies urgency label thresholds: ≥8 → "CRITICAL", ≥6 → "High", ≥4 → "Medium", else → "Low". |
| | • Selects Slack channel: Billing → `slackBillingChannel`, Technical → `slackTechChannel`, General (or unknown) → `slackSupportChannel`. |
| | • Formats `receivedAt` to `YYYY‑MM‑DD HH:mm` format. |
| | • Generates `loggedAt` timestamp in the same format. |
| | • Falls back to sensible defaults if AI fields are missing (e.g. `department` defaults to `"General"`, `urgencyScore` defaults to `0`). |
| **Key Expressions** | `$('3. Code — Extract Email Fields').item.json` for referencing earlier node data; date formatting via `new Date().toISOString().replace('T', ' ').substring(0, 16)`. |
| **Credential** | None. |
| **Input** | Output of node 5 (AI Agent). |
| **Output** | Flat JSON with 15 keys: `emailId`, `senderEmail`, `senderName`, `subject`, `emailBody`, `receivedDate`, `supportEmail`, `sheetId`, `sheetName`, `urgencyScore`, `urgencyLabel`, `department`, `oneLineSummary`, `suggestedReply`, `slackChannel`, `loggedAt`. |
| **Edge Cases / Failure Types** | • If the AI agent failed to produce output (node 5 error), this node will also fail because `$input.first().json.output` is undefined. • Unknown department values (not Billing/Technical/General) default to General and route to the general support channel. • Timestamps rely on the system clock of the n8n instance. |

---

### Block 1.5 — Output — Logging & Alerting

**Overview:** This block writes a permanent, 12‑column record of every complaint to Google Sheets and simultaneously posts a richly formatted Slack alert to the department‑specific channel. Both actions execute in parallel, ensuring low latency from email receipt to team notification.

**Nodes Involved:**
- 9. Google Sheets — Log Complaint
- 10. Slack — Send Department Alert

---

#### Node: 9. Google Sheets — Log Complaint

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.googleSheets` (v4.5) |
| **Functional Role** | Appends a single row to the "Complaint Log" tab in the configured Google Sheet, creating a permanent, searchable record of every triaged complaint. |
| **Configuration** | • `operation`: append • `documentId`: dynamic — `{{ $json.sheetId }}` (from config) • `sheetName`: dynamic — `{{ $json.sheetName }}` (from config, defaults to "Complaint Log") • `cellFormat`: USER_ENTERED (allows Sheets to interpret dates/formulas) • Column mapping (12 columns defined below): |
| | 1. `Email ID` ← `{{ $json.emailId }}` |
| | 2. `Received Date` ← `{{ $json.receivedDate }}` |
| | 3. `Sender Name` ← `{{ $json.senderName }}` |
| | 4. `Sender Email` ← `{{ $json.senderEmail }}` |
| | 5. `Subject` ← `{{ $json.subject }}` |
| | 6. `One Line Summary` ← `{{ $json.oneLineSummary }}` |
| | 7. `Department` ← `{{ $json.department }}` |
| | 8. `Urgency Score` ← `{{ $json.urgencyScore }}` |
| | 9. `Urgency Level` ← `{{ $json.urgencyLabel }}` |
| | 10. `Suggested Reply` ← `{{ $json.suggestedReply }}` |
| | 11. `Slack Channel Alerted` ← `{{ $json.slackChannel }}` |
| | 12. `Logged At` ← `{{ $json.loggedAt }}` |
| **Credential** | Google Sheets OAuth2 — must have edit access to the target spreadsheet. |
| **Input** | Output of node 8. |
| **Output** | Confirmation of appended row (not consumed further). |
| **Edge Cases / Failure Types** | • Incorrect `sheetId` → "Spreadsheet not found" error. • Tab name mismatch (e.g. "complaint log" vs "Complaint Log") → "Sheet not found" error. • Missing column headers in row 1 → data will still be appended but columns may be misaligned. • OAuth2 token expiration → re‑authorise the credential. • Rate limiting on Sheets API → n8n retries automatically. |

---

#### Node: 10. Slack — Send Department Alert

| Property | Detail |
|---|---|
| **Type** | `n8n-nodes-base.slack` (v2.2) |
| **Functional Role** | Posts a formatted alert to the appropriate Slack channel, displaying urgency, department, sender details, summary, and the suggested reply for agents to copy. |
| **Configuration** | • `authentication`: OAuth2 • `channel`: dynamic — `{{ $json.slackChannel }}` (selected in node 8 based on department) • `text`: Richly formatted message (see below) • `otherOptions.mrkdwn`: true (enables Slack markdown formatting) |
| **Message Template** | |
| | `*NEW COMPLAINT ALERT — {{ $json.urgencyLabel }}*` |
| | `*Urgency Score:* {{ $json.urgencyScore }}/10` |
| | `*Department:* {{ $json.department }}` |
| | `*From:* {{ $json.senderName }} ({{ $json.senderEmail }})` |
| | `*Subject:* {{ $json.subject }}` |
| | `*Received:* {{ $json.receivedDate }}` |
| | `*Summary:*` |
| | `{{ $json.oneLineSummary }}` |
| | `*Suggested Reply for Agent:*` |
| | `{{ $json.suggestedReply }}` |
| | `---` |
| | `_Reply to: {{ $json.senderEmail }} \| Support: {{ $json.supportEmail }}_` |
| | `_Logged to Google Sheets automatically_` |
| **Credential** | Slack OAuth2 — the bot must be invited to all three target channels. |
| **Input** | Output of node 8. |
| **Output** | Confirmation of Slack message post (not consumed further). |
| **Edge Cases / Failure Types** | • Bot not invited to channel → "channel_not_found" or "not_in_channel" error. • Channel name typo or missing `#` prefix → same error. • OAuth2 token revoked → re‑authorise. • Message longer than Slack's 40 000 character limit (extremely unlikely). • Special characters in sender name or subject may need escaping (handled by Slack's API). |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1. Gmail — Inbox Monitor | gmailTrigger | Entry trigger; polls Gmail every 60 seconds for new emails | — | 2. Set — Config Values | **Section — Inbox Monitoring and Config**: Gmail is polled every minute for new emails. Config stores company name, Sheet ID, three Slack channel names, and support email — all set once here. |
| 2. Set — Config Values | set | Central configuration store for seven reusable values | 1. Gmail — Inbox Monitor | 3. Code — Extract Email Fields | **Section — Inbox Monitoring and Config**: Gmail is polled every minute for new emails. Config stores company name, Sheet ID, three Slack channel names, and support email — all set once here. |
| 3. Code — Extract Email Fields | code | Parses Gmail payload, extracts sender/subject/body, truncates body, merges config values | 2. Set — Config Values | 4. IF — Is This a Complaint? | **Section — Email Parsing and Complaint Detection**: Extracts sender details, subject, and body from the raw Gmail payload. Scans subject and body for 12 complaint keywords. Non-complaint emails exit cleanly via the NoOp node. |
| 4. IF — Is This a Complaint? | if | Keyword filter; TRUE path continues to AI triage, FALSE path exits silently | 3. Code — Extract Email Fields | 5. AI Agent — Triage Complaint (TRUE), 11. NoOp — Not a Complaint (FALSE) | **Section — Email Parsing and Complaint Detection**: Extracts sender details, subject, and body from the raw Gmail payload. Scans subject and body for 12 complaint keywords. Non-complaint emails exit cleanly via the NoOp node. |
| 11. NoOp — Not a Complaint | noOp | Silent terminal for non‑complaint emails | 4. IF — Is This a Complaint? (FALSE) | — | **Section — Email Parsing and Complaint Detection**: Extracts sender details, subject, and body from the raw Gmail payload. Scans subject and body for 12 complaint keywords. Non-complaint emails exit cleanly via the NoOp node. |
| 5. AI Agent — Triage Complaint | @n8n/n8n-nodes-langchain.agent | Orchestrates GPT‑4o‑mini call to score urgency, classify department, summarise, and draft a reply | 4. IF — Is This a Complaint? (TRUE) | 8. Code — Combine Triage Data | **Section — AI Triage Analysis**: GPT-4o-mini returns four structured fields: urgency score (1–10), department (Billing, Technical, or General), one-line summary, and a ready-to-send reply draft. Structured Output Parser enforces the schema. |
| 6. OpenAI — GPT‑4o‑mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model backend for the AI Agent | — (connected to node 5 via ai_languageModel) | 5. AI Agent — Triage Complaint (ai_languageModel) | **Section — AI Triage Analysis**: GPT-4o-mini returns four structured fields: urgency score (1–10), department (Billing, Technical, or General), one-line summary, and a ready-to-send reply draft. Structured Output Parser enforces the schema. |
| 7. Parser — Structured Triage Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema on AI output (urgencyScore, department, oneLineSummary, suggestedReply) | — (connected to node 5 via ai_outputParser) | 5. AI Agent — Triage Complaint (ai_outputParser) | **Section — AI Triage Analysis**: GPT-4o-mini returns four structured fields: urgency score (1–10), department (Billing, Technical, or General), one-line summary, and a ready-to-send reply draft. Structured Output Parser enforces the schema. |
| 8. Code — Combine Triage Data | code | Merges AI output with email data; computes urgency label and Slack channel; adds timestamp | 5. AI Agent — Triage Complaint | 9. Google Sheets — Log Complaint, 10. Slack — Send Department Alert | **Section — Data Assembly**: Merges AI output with original email data. Sets urgency label (Low, Medium, High, CRITICAL) based on score. Routes to the correct Slack channel based on department. |
| 9. Google Sheets — Log Complaint | googleSheets | Appends a 12‑column row to the Complaint Log sheet | 8. Code — Combine Triage Data | — | **Section — Complaint Log and Department Alert**: Google Sheets appends a 12-column permanent record for every complaint. Slack posts a formatted alert with urgency, summary, and reply draft to the correct department channel. |
| 10. Slack — Send Department Alert | slack | Posts a formatted alert to the department‑specific Slack channel | 8. Code — Combine Triage Data | — | **Section — Complaint Log and Department Alert**: Google Sheets appends a 12-column permanent record for every complaint. Slack posts a formatted alert with urgency, summary, and reply draft to the correct department channel. |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps in order to recreate the entire workflow in a new n8n instance.

### Prerequisites

1. **n8n instance** (self‑hosted or cloud) with internet access.
2. **Gmail account** — the inbox to monitor.
3. **OpenAI account** with API access to the `gpt-4o-mini` model.
4. **Slack workspace** with an OAuth2 app and three public channels (e.g. `#billing-support`, `#tech-support`, `#customer-support`).
5. **Google Sheets** — a spreadsheet with a tab named **Complaint Log** and the following 12 column headers in row 1: `Email ID`, `Received Date`, `Sender Name`, `Sender Email`, `Subject`, `One Line Summary`, `Department`, `Urgency Score`, `Urgency Level`, `Suggested Reply`, `Slack Channel Alerted`, `Logged At`.

### Step‑by‑Step Instructions

1. **Create a new workflow** in n8n and name it "Customer Complaint Triage".

2. **Add the Gmail Trigger node**
   - Type: `Gmail Trigger` (`n8n-nodes-base.gmailTrigger`), version 1.3.
   - Settings: `simple` = false, `pollTimes` = every minute, no label/query filters.
   - Credential: Add a Gmail OAuth2 credential and authorise read‑only access to the target inbox.

3. **Add the Set node — Config Values**
   - Type: `Set` (`n8n-nodes-base.set`), version 3.4.
   - Add seven string assignments:
     - `companyName` → your business name (e.g. "Acme Corp")
     - `sheetId` → your Google Sheet ID (the string between `/d/` and `/edit` in the Sheet URL)
     - `sheetName` → `Complaint Log` (must match your tab name exactly)
     - `slackBillingChannel` → `#billing-support` (or your billing channel)
     - `slackTechChannel` → `#tech-support` (or your tech channel)
     - `slackSupportChannel` → `#customer-support` (or your general channel)
     - `supportEmail` → your support team email (e.g. `support@acme.com`)
   - Connect: Gmail Trigger → Set (main output to main input).

4. **Add the Code node — Extract Email Fields**
   - Type: `Code` (`n8n-nodes-base.code`), version 2.
   - Paste the JavaScript code from the workflow (or replicate the logic):
     - Extract `senderEmail` and `senderName` from `from.text` using regex.
     - Extract `subject` and `emailBody` from the Gmail payload, falling back to `snippet`.
     - Truncate `emailBody` to 3 000 characters.
     - Read `receivedAt` from `date` and `emailId` from `id`/`messageId`.
     - Merge in all seven config values from the Set node using `$('2. Set — Config Values').item.json`.
   - Connect: Set → Code (Extract Email Fields).

5. **Add the IF node — Is This a Complaint?**
   - Type: `IF` (`n8n-nodes-base.if`), version 2.2.
   - Condition type: single boolean expression, case‑insensitive.
   - Expression checks 12 keywords across `subject` and `emailBody`:
     - Subject: `complaint`, `issue`, `problem`, `not working`, `broken`, `refund`, `angry`, `disappointed`
     - Body: `complaint`, `very unhappy`, `unacceptable`, `demand refund`, `worst`
   - Connect: Code (Extract Email Fields) → IF.
   - TRUE output → AI Agent (step 6).
   - FALSE output → NoOp (step 7).

6. **Add the NoOp node — Not a Complaint**
   - Type: `NoOp` (`n8n-nodes-base.noOp`), version 1.
   - No configuration needed.
   - Connect: IF (FALSE branch) → NoOp.

7. **Add the AI Agent node — Triage Complaint**
   - Type: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`), version 1.7.
   - `promptType`: define.
   - `hasOutputParser`: true.
   - `text` (prompt): Copy the full prompt from the workflow. Key elements:
     - Role: "You are a customer support triage specialist at `{{ $json.companyName }}`."
     - Input: sender name, sender email, subject, email body.
     - Required output: JSON with four fields — `urgencyScore` (1–10), `department` (Billing/Technical/General), `oneLineSummary` (≤20 words), `suggestedReply` (≤80 words, start with customer name, end with company name).
     - Rules: return only JSON, no markdown, urgencyScore must be a number, department must be exactly one of the three values.
   - Connect: IF (TRUE branch) → AI Agent (main input).

8. **Add the OpenAI Model node — GPT‑4o‑mini**
   - Type: `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), version 1.2.
   - Settings: `model` = `gpt-4o-mini`, `temperature` = 0.3, `maxTokens` = 600.
   - Credential: Add an OpenAI API key credential with access to `gpt-4o-mini`.
   - Connect: OpenAI Model → AI Agent (ai_languageModel input).

9. **Add the Structured Output Parser node**
   - Type: `Structured Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`), version 1.3.
   - `schemaType`: manual.
   - `inputSchema`: Paste the JSON schema with four required properties:
     - `urgencyScore` (number, 1–10)
     - `department` (string, Billing/Technical/General)
     - `oneLineSummary` (string)
     - `suggestedReply` (string)
   - Connect: Parser → AI Agent (ai_outputParser input).

10. **Add the Code node — Combine Triage Data**
    - Type: `Code` (`n8n-nodes-base.code`), version 2.
    - Paste the JavaScript code from the workflow (or replicate):
      - Read AI output from `$input.first().json.output`.
      - Read original email data from `$('3. Code — Extract Email Fields').item.json`.
      - Apply urgency label thresholds (≥8 → CRITICAL, ≥6 → High, ≥4 → Medium, else → Low).
      - Select Slack channel based on department (Billing/Technical/General).
      - Format `receivedDate` and add `loggedAt` timestamp.
      - Fall back to defaults for missing AI fields.
    - Connect: AI Agent → Code (Combine Triage Data).

11. **Add the Google Sheets node — Log Complaint**
    - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.5.
    - Settings: `operation` = append, `documentId` = `{{ $json.sheetId }}`, `sheetName` = `{{ $json.sheetName }}`, `cellFormat` = USER_ENTERED.
    - Column mapping (12 columns):
      - `Email ID` ← `{{ $json.emailId }}`
      - `Received Date` ← `{{ $json.receivedDate }}`
      - `Sender Name` ← `{{ $json.senderName }}`
      - `Sender Email` ← `{{ $json.senderEmail }}`
      - `Subject` ← `{{ $json.subject }}`
      - `One Line Summary` ← `{{ $json.oneLineSummary }}`
      - `Department` ← `{{ $json.department }}`
      - `Urgency Score` ← `{{ $json.urgencyScore }}`
      - `Urgency Level` ← `{{ $json.urgencyLabel }}`
      - `Suggested Reply` ← `{{ $json.suggestedReply }}`
      - `Slack Channel Alerted` ← `{{ $json.slackChannel }}`
      - `Logged At` ← `{{ $json.loggedAt }}`
    - Credential: Add a Google Sheets OAuth2 credential with edit access to the target spreadsheet.
    - Connect: Code (Combine Triage Data) → Google Sheets (main output, index 0).

12. **Add the Slack node — Send Department Alert**
    - Type: `Slack` (`n8n-nodes-base.slack`), version 2.2.
    - Settings: `authentication` = OAuth2, `channel` = `{{ $json.slackChannel }}`, `otherOptions.mrkdwn` = true.
    - `text`: Copy the formatted message template from the workflow (includes urgency label, score, department, sender details, subject, summary, suggested reply, and footer with reply‑to and support email).
    - Credential: Add a Slack OAuth2 credential and invite the n8n bot to all three target channels (`/invite @n8n` in each channel).
    - Connect: Code (Combine Triage Data) → Slack (main output, index 0).

13. **Verify all connections**
    - Gmail Trigger → Set → Code (Extract) → IF → (TRUE) AI Agent → Code (Combine) → [Google Sheets, Slack]
    - IF → (FALSE) NoOp
    - OpenAI Model → AI Agent (ai_languageModel)
    - Parser → AI Agent (ai_outputParser)

14. **Activate the workflow**
    - Toggle the workflow to **Active**.
    - Gmail will be polled every 60 seconds; new complaint emails will trigger the full triage pipeline automatically.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Customer Complaint Triage workflow monitors Gmail every minute, uses GPT‑4o‑mini for AI scoring and routing, logs to Google Sheets, and alerts Slack channels automatically. | Overall workflow purpose |
| The email body is truncated to 3 000 characters in the Code — Extract Email Fields node to keep OpenAI token usage predictable (≈600 max output tokens). | Token efficiency design choice |
| Urgency label thresholds (≥8 CRITICAL, ≥6 High, ≥4 Medium, else Low) are defined in the Code — Combine Triage Data node and can be adjusted without touching any other node. | Customisation point |
| The 12 complaint keywords (complaint, issue, problem, not working, broken, refund, angry, disappointed, very unhappy, unacceptable, demand refund, worst) are embedded in the IF node expression and can be extended inline. | Customisation point |
| Non‑complaint emails exit via the NoOp node with zero API calls, zero logging, and zero Slack notifications — ensuring cost efficiency. | Noise reduction |
| The Structured Output Parser enforces a strict JSON schema on GPT‑4o‑mini's response, preventing malformed data from reaching Sheets or Slack. | Reliability feature |
| Support contact for customisation or help: info@incrementors.com | https://www.incrementors.com/contact-us/ |
| All credentials (Gmail OAuth2, OpenAI API key, Slack OAuth2, Google Sheets OAuth2) must be configured before activating the workflow. | Setup prerequisite |
| The Google Sheet must contain a tab named exactly "Complaint Log" (case‑sensitive) with 12 column headers in row 1 before the workflow runs for the first time. | Setup prerequisite |
| The n8n Slack bot must be invited to all three target channels (`#billing-support`, `#tech-support`, `#customer-support`) before the Slack node can post messages. | Setup prerequisite |