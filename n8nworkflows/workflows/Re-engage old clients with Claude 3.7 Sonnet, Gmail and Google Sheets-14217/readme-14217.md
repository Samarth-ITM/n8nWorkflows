Re-engage old clients with Claude 3.7 Sonnet, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/re-engage-old-clients-with-claude-3-7-sonnet--gmail-and-google-sheets-14217


# Re-engage old clients with Claude 3.7 Sonnet, Gmail and Google Sheets

# 1. Workflow Overview

This workflow automates daily re-engagement of former clients using Google Sheets as the client database, Gmail as both the email-history source and sending channel, and Anthropic Claude 3.7 Sonnet to draft personalized follow-up emails.

Its main objective is to contact inactive clients without spamming those who recently replied. It checks recent client replies, updates operational status in the sheet, gathers contextual templates and conversation history, generates a structured AI email draft, sends the email, updates counters, and loops to the next client.

## 1.1 Trigger & Client Loading

The workflow starts on a daily schedule and reads client rows from a Google Sheet used as the outreach database. This acts as the master input source for all downstream logic.

## 1.2 Per-Client Iteration

Loaded rows are split into single items so each client is processed independently. The workflow returns to the loop after each client is completed or skipped.

## 1.3 Recent Reply Detection

For each client, the workflow fetches the latest email from Gmail, calculates whether the client replied within the last 30 days, and formats useful date metadata for routing and sheet updates.

## 1.4 Reply-Based Routing

Clients who replied recently are marked for manual handling and skipped. Clients without a recent reply proceed into the AI-driven outreach branch.

## 1.5 Context Collection for AI Drafting

For clients eligible for follow-up, the workflow marks them as actively being processed, retrieves a follow-up template, retrieves a situation-specific prompt, and fetches the last 10 Gmail conversations for context.

## 1.6 AI Draft Generation

The workflow merges prompt/template/history inputs, aggregates them, and sends a composite prompt to an n8n LangChain agent backed by Claude 3.7 Sonnet. A structured output parser enforces a JSON-like email schema.

## 1.7 Email Sending, Logging, and Loop Continuation

The drafted email is sent via Gmail, the number of sent emails is incremented in Google Sheets, the workflow waits one minute for rate limiting, and then returns to the loop for the next client.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Load Clients

### Overview

This block launches the workflow once per day and loads client records from the Google Sheets database. It is the entry point and defines the set of clients to process in the current run.

### Nodes Involved

- ⏰ Daily Schedule Trigger
- 📋 Old Client Database

### Node Details

#### 1) ⏰ Daily Schedule Trigger

- **Type / role:** `n8n-nodes-base.scheduleTrigger` — scheduled entry point.
- **Configuration:** Uses an interval-based rule with default interval settings. In practice, this represents a recurring daily run, though the exact cadence should be verified in the node UI because the JSON does not specify a custom cron expression.
- **Key variables / expressions:** None.
- **Input connections:** None; this is an entry node.
- **Output connections:** Sends execution to **📋 Old Client Database**.
- **Version-specific notes:** Type version `1.2`.
- **Potential failures / edge cases:**
  - Workflow inactive, so no schedule execution occurs.
  - Timezone mismatch if the instance timezone differs from business expectations.
  - Misconfigured interval if the node was not finalized in the editor.

#### 2) 📋 Old Client Database

- **Type / role:** `n8n-nodes-base.googleSheets` — reads the client database from Google Sheets.
- **Configuration:** Reads from spreadsheet `YOUR_GOOGLE_SHEET_ID`, sheet/tab `gid=0` named **Database**. A filter is configured on the column **Manually Stop Workflow (STOP)**, intended to support manual override from the sheet.
- **Key variables / expressions:** No dynamic filter value is shown, only a lookup column. This suggests the intent is filtering by presence/absence or a manually set condition, but this should be verified because the current configuration may not fully enforce exclusion on its own.
- **Input connections:** From **⏰ Daily Schedule Trigger**.
- **Output connections:** To **🔄 Loop: Process Each Client**.
- **Version-specific notes:** Type version `4.5`.
- **Potential failures / edge cases:**
  - Google Sheets OAuth not connected.
  - Wrong spreadsheet ID or inaccessible sheet.
  - Column names not exactly matching the sheet.
  - Filter configuration may not behave as intended if no lookup value is defined.
  - Empty sheet results in no downstream processing.

---

## Block 2 — Loop: Process Each Client

### Overview

This block converts the loaded list of clients into one-at-a-time processing. It also serves as the return point after each client is skipped or fully processed.

### Nodes Involved

- 🔄 Loop: Process Each Client

### Node Details

#### 1) 🔄 Loop: Process Each Client

- **Type / role:** `n8n-nodes-base.splitInBatches` — iterative batch processor.
- **Configuration:** Default options; processes one item at a time unless changed in UI.
- **Key variables / expressions:** None.
- **Input connections:** From **📋 Old Client Database**, and loop-back connections from **📝 Update: Mark Reply Manually** and **⏱️ Rate Limit Wait (1 Min)**.
- **Output connections:**
  - Output 1 continues iteration to **📧 Fetch Client's Latest Email**.
  - Output 0 is the completion path and is unused.
- **Version-specific notes:** Type version `3`.
- **Potential failures / edge cases:**
  - If upstream returns no rows, no items will be processed.
  - Incorrect loop-back wiring can cause infinite loops or skipped items.
  - Long-running executions may accumulate if the client list is large.

---

## Block 3 — Check Latest Email from Client

### Overview

This block retrieves the most recent Gmail message from the client and computes whether that message is recent enough to suppress automated outreach. It also prepares formatted date values for storage.

### Nodes Involved

- 📧 Fetch Client's Latest Email
- 📅 Date Checker (30-Day Window)

### Node Details

#### 1) 📧 Fetch Client's Latest Email

- **Type / role:** `n8n-nodes-base.gmail` — fetches the latest email from Gmail matching the client’s email address.
- **Configuration:** Operation `getAll`, limit `1`, `simple=false`, filter `sender = {{$json['Email Address']}}`.
- **Key variables / expressions:**
  - `={{ $json['Email Address'] }}`
- **Input connections:** From **🔄 Loop: Process Each Client**.
- **Output connections:** To **📅 Date Checker (30-Day Window)**.
- **Version-specific notes:** Type version `2.1`.
- **Potential failures / edge cases:**
  - Gmail OAuth missing or expired.
  - No matching messages found; downstream code must handle empty input.
  - The sender filter may miss messages if aliases, forwarding, or differing sender headers are involved.
  - `simple=false` means rawer output fields are used; downstream depends specifically on `date` and `text`.

#### 2) 📅 Date Checker (30-Day Window)

- **Type / role:** `n8n-nodes-base.code` — computes recency metadata.
- **Configuration:** JavaScript loops through all input items and adds:
  - `isWithinLast30Days`
  - `daysDifference`
  - `thirtyDaysAgo`
  - `formattedDate`
- **Key logic:**
  - Uses `item.json.date || '2024-09-24T02:52:24.000Z'`
  - Compares input date to `Date.now() - 30 days`
  - Formats date as `D-Mon-YYYY`
- **Input connections:** From **📧 Fetch Client's Latest Email**.
- **Output connections:** To **🔀 Route: Client Replied Recently?**
- **Version-specific notes:** Type version `2`.
- **Potential failures / edge cases:**
  - If Gmail returns no items, this node receives nothing and no route decision is made for that client.
  - The fallback hardcoded date (`2024-09-24T02:52:24.000Z`) can create misleading behavior if `date` is missing.
  - Invalid or unexpected date formats can produce `Invalid Date`.
  - `daysDifference` can be negative if a malformed future date is returned.

---

## Block 4 — Route by Reply Status

### Overview

This decision block routes clients based on whether they replied within the last 30 days. It is the primary safeguard against sending automated emails to recently active clients.

### Nodes Involved

- 🔀 Route: Client Replied Recently?

### Node Details

#### 1) 🔀 Route: Client Replied Recently?

- **Type / role:** `n8n-nodes-base.switch` — boolean branching.
- **Configuration:** Two rules:
  - Route 0: `isWithinLast30Days === true`
  - Route 1: `isWithinLast30Days === false`
  - `allMatchingOutputs = true`
- **Key variables / expressions:**
  - `={{ $json.isWithinLast30Days }}`
- **Input connections:** From **📅 Date Checker (30-Day Window)**.
- **Output connections:**
  - Route 0 → **📝 Update: Mark Reply Manually**
  - Route 1 → **📂 Fetch Situation-Based Prompt**, **📩 Fetch Followup Message Template**, **✅ Update: Mark Running Status**
- **Version-specific notes:** Type version `3.2`.
- **Potential failures / edge cases:**
  - If `isWithinLast30Days` is undefined, neither route may behave as expected.
  - Strict type validation means strings like `"true"` will not match boolean `true`.
  - If no email item exists, this node is never reached.

---

## Block 5 — Client Replied Recently: Mark and Skip

### Overview

This branch updates the client row with the latest client email content and date, marks the workflow status as requiring manual reply, and returns to the loop without sending anything.

### Nodes Involved

- 📝 Update: Mark Reply Manually

### Node Details

#### 1) 📝 Update: Mark Reply Manually

- **Type / role:** `n8n-nodes-base.googleSheets` — updates the Database sheet for recently active clients.
- **Configuration:** Update operation on the **Database** sheet, matching by **Email Address**. Writes:
  - `Email Address` from the original database row
  - `Lastest Email from Client` from fetched Gmail `text`
  - `Date of Lastest Email from Client` from formatted date
  - `Followup Workflow (Running,Reply Manually)` = `Reply Manually`
- **Key variables / expressions:**
  - `={{ $('📋 Old Client Database').item.json['Email Address'] }}`
  - `={{ $('📧 Fetch Client\'s Latest Email').item.json.text }}`
  - `={{ $('📅 Date Checker (30-Day Window)').item.json.formattedDate }}`
- **Input connections:** From route 0 of **🔀 Route: Client Replied Recently?**
- **Output connections:** Loops back to **🔄 Loop: Process Each Client**
- **Version-specific notes:** Type version `4.6`.
- **Potential failures / edge cases:**
  - Matching fails if email addresses are duplicated or formatted inconsistently.
  - The sheet column label contains the typo `Lastest`, which must match exactly in Google Sheets.
  - If Gmail text is empty or too long, sheet update may be incomplete or truncated.
  - If upstream item linking breaks, the `$()` references can fail.

---

## Block 6 — Prepare Outreach Context

### Overview

This block is executed only for clients who have not replied recently. It marks processing status, fetches a follow-up message template and a situation prompt from Google Sheets, and retrieves up to 10 prior email conversations from Gmail.

### Nodes Involved

- ✅ Update: Mark Running Status
- 📩 Fetch Followup Message Template
- 📂 Fetch Situation-Based Prompt
- 📬 Fetch 10 Email Conversations
- 📦 Bundle Email Conversations

### Node Details

#### 1) ✅ Update: Mark Running Status

- **Type / role:** `n8n-nodes-base.googleSheets` — sets the outreach state to active.
- **Configuration:** Update on **Database** sheet, matching by **Email Address**. Writes:
  - `Followup Workflow (Running,Reply Manually)` = `Running`
  - Resets latest email text/date fields to `=`
- **Key variables / expressions:**
  - `={{ $('📋 Old Client Database').item.json['Email Address'] }}`
- **Input connections:** From route 1 of **🔀 Route: Client Replied Recently?**
- **Output connections:** To **📬 Fetch 10 Email Conversations**
- **Version-specific notes:** Type version `4.6`.
- **Potential failures / edge cases:**
  - Writing literal `=` into fields may be intentional clearing, but in Sheets it may be treated oddly depending on parsing.
  - Duplicate email matches may update the wrong row if the sheet is not unique.

#### 2) 📩 Fetch Followup Message Template

- **Type / role:** `n8n-nodes-base.googleSheets` — retrieves a matching follow-up direction/template.
- **Configuration:** Reads from tab **Follow-up Messages** (`gid=1592686330`) using filters:
  - `Scenario = {{$('📋 Old Client Database').item.json.Scenario}}`
  - `Number of Emails Sent = {{$('📋 Old Client Database').item.json['Number of Emails Sent']}}`
- **Error handling:** `onError = continueErrorOutput`
- **Input connections:** From route 1 of **🔀 Route: Client Replied Recently?**
- **Output connections:** To **🔀 Merge: Templates + Conversations**
- **Version-specific notes:** Type version `4.5`.
- **Potential failures / edge cases:**
  - Missing template row for the scenario/email-count combination.
  - Because errors continue, downstream logic may receive empty data instead of stopping.
  - Sheet values must match exactly, including numeric-vs-string representations.

#### 3) 📂 Fetch Situation-Based Prompt

- **Type / role:** `n8n-nodes-base.googleSheets` — retrieves a scenario-specific prompt.
- **Configuration:** Reads from tab **Situation** (`gid=666858875`) filtered by:
  - `Scenario = {{$('📋 Old Client Database').item.json.Scenario}}`
- **Error handling:** `onError = continueErrorOutput`
- **Input connections:** From route 1 of **🔀 Route: Client Replied Recently?**
- **Output connections:** To **🔀 Merge: Templates + Conversations**
- **Version-specific notes:** Type version `4.5`.
- **Potential failures / edge cases:**
  - No prompt found for the scenario.
  - Since execution continues on error, later nodes may draft with incomplete context.
  - Exact column naming and data typing matter.

#### 4) 📬 Fetch 10 Email Conversations

- **Type / role:** `n8n-nodes-base.gmail` — retrieves recent email history for context.
- **Configuration:** Operation `getAll`, limit `10`, `simple=false`, filter by sender email address from current item.
- **Key variables / expressions:**
  - `={{ $json['Email Address'] }}`
- **Input connections:** From **✅ Update: Mark Running Status**
- **Output connections:** To **📦 Bundle Email Conversations**
- **Version-specific notes:** Type version `2.1`.
- **Potential failures / edge cases:**
  - No matching messages found.
  - Sender-only filtering may omit messages where the client was recipient rather than sender.
  - Gmail rate limits may apply on larger runs.

#### 5) 📦 Bundle Email Conversations

- **Type / role:** `n8n-nodes-base.aggregate` — aggregates multiple email texts into one item.
- **Configuration:** Aggregates field `text`.
- **Input connections:** From **📬 Fetch 10 Email Conversations**
- **Output connections:** To input 2 of **🔀 Merge: Templates + Conversations**
- **Version-specific notes:** Type version `1`.
- **Potential failures / edge cases:**
  - If no emails are fetched, aggregate may output an empty structure or no item depending on runtime behavior.
  - Large text histories may make prompts too long for the LLM.

---

## Block 7 — Merge & Prepare AI Input

### Overview

This block combines the follow-up template, situation prompt, and bundled conversation history into a single AI-ready payload.

### Nodes Involved

- 🔀 Merge: Templates + Conversations
- 📦 Prepare AI Input Bundle

### Node Details

#### 1) 🔀 Merge: Templates + Conversations

- **Type / role:** `n8n-nodes-base.merge` — merges three upstream streams.
- **Configuration:** `numberInputs = 3`
- **Input connections:**
  - Input 0: **📩 Fetch Followup Message Template**
  - Input 1: **📂 Fetch Situation-Based Prompt**
  - Input 2: **📦 Bundle Email Conversations**
- **Output connections:** To **📦 Prepare AI Input Bundle**
- **Version-specific notes:** Type version `3.1`.
- **Potential failures / edge cases:**
  - If one input emits no item, merge behavior may not produce the expected combined record.
  - Error-tolerant sheet nodes can pass partial or missing context into this merge.

#### 2) 📦 Prepare AI Input Bundle

- **Type / role:** `n8n-nodes-base.aggregate` — aggregates relevant fields into one bundle for prompting.
- **Configuration:** Aggregates:
  - `Prompt`
  - `text`
  - `Followup Message Direction`
- **Input connections:** From **🔀 Merge: Templates + Conversations**
- **Output connections:** To **🤖 Re-engagement Email Drafter**
- **Version-specific notes:** Type version `1`.
- **Potential failures / edge cases:**
  - Missing `Prompt` or `Followup Message Direction` leads to weak or malformed prompts.
  - Aggregated arrays/values may not be formatted exactly as the agent expects.

---

## Block 8 — AI Drafts and Sends Email

### Overview

This block uses Claude 3.7 Sonnet through the LangChain Agent node to generate a structured re-engagement email, validates the output format with a structured parser, and sends the result via Gmail.

### Nodes Involved

- 🤖 Re-engagement Email Drafter
- 🤖 Claude 3.7 Sonnet LLM
- 📋 Email Structure Parser
- 📤 Send Re-engagement Email

### Node Details

#### 1) 🤖 Re-engagement Email Drafter

- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — AI agent that creates the re-engagement email.
- **Configuration:**
  - Prompt is explicitly defined.
  - Uses a system message:  
    `Draft an email based on the recent conversation done with the client convincing him that, we have now improved and help him achieve his goals.`
  - Requires an output parser.
- **Key variables / expressions in prompt:**
  - `{{ $json.Prompt }}`
  - `{{ $('📩 Fetch Followup Message Template').item.json['Followup Message Direction'] }}`
  - `{{ $json.text }}`
  - `{{ $('📋 Old Client Database').item.json["Email Address"] }}`
  - `{{ $('📋 Old Client Database').item.json['The goal which client wanted to achieve by hiring an agency'] }}`
  - `{{ $('📋 Old Client Database').item.json['Detailed Reason Why Contract Got Ended'] }}`
  - `{{ $('📋 Old Client Database').item.json['Client is from Industry of'] }}`
  - `{{ $('📋 Old Client Database').item.json['Client\'s Industry Vertical'] }}`
  - `{{ $('📋 Old Client Database').item.json['Specific services they used'] }}`
- **Input connections:** Main input from **📦 Prepare AI Input Bundle**; AI model input from **🤖 Claude 3.7 Sonnet LLM**; parser input from **📋 Email Structure Parser**.
- **Output connections:** To **📤 Send Re-engagement Email**
- **Version-specific notes:** Type version `1.9`.
- **Potential failures / edge cases:**
  - Missing sheet columns used in expressions will break prompt construction.
  - Token/context limits may be reached if email history is large.
  - If template/prompt lookups returned empty data, output quality may degrade sharply.
  - Anthropic API/auth errors can stop execution.

#### 2) 🤖 Claude 3.7 Sonnet LLM

- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — Anthropic chat model provider.
- **Configuration:** Model set to `claude-3-7-sonnet-20250219`.
- **Input connections:** None on main channel; connected as AI language model to **🤖 Re-engagement Email Drafter**.
- **Output connections:** AI language model output to the agent.
- **Version-specific notes:** Type version `1.3`.
- **Potential failures / edge cases:**
  - Anthropic credentials missing or invalid.
  - Model availability differs by account/region.
  - API quota/rate-limit issues.

#### 3) 📋 Email Structure Parser

- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured AI output.
- **Configuration:** Manual JSON schema requiring:
  - `subject` string
  - `body` object with `greeting`, `content`, `closing`
  - Required fields: `subject`, `body`, and `body.content`
- **Input connections:** None on main channel; connected as AI output parser to **🤖 Re-engagement Email Drafter**.
- **Output connections:** Parser output into the agent.
- **Version-specific notes:** Type version `1.2`.
- **Potential failures / edge cases:**
  - If the model fails to comply with the schema, the agent may error.
  - `greeting` and `closing` are not required, so downstream email body may omit them.
  - The send node currently uses greeting and content but ignores closing.

#### 4) 📤 Send Re-engagement Email

- **Type / role:** `n8n-nodes-base.gmail` — sends the drafted email.
- **Configuration:**
  - `sendTo = user@example.com`
  - Subject from `{{$json.output.subject}}`
  - Message from `{{$json.output.body.greeting}}\n\n{{$json.output.body.content}}`
  - Email type `text`
  - Attribution disabled
- **Important note:** The recipient is hardcoded to `user@example.com` and does **not** currently use the client’s email address dynamically. This must be changed for production.
- **Input connections:** From **🤖 Re-engagement Email Drafter**
- **Output connections:** To **📊 Update Email Count in Sheet**
- **Version-specific notes:** Type version `2.1`.
- **Potential failures / edge cases:**
  - Gmail OAuth/auth failure.
  - Hardcoded recipient causes all emails to go to the placeholder address unless replaced.
  - Missing `output.subject` or `output.body.content` will break sending.
  - Closing is generated but not included in the sent body.

---

## Block 9 — Update Count, Wait, and Loop Back

### Overview

After sending, this block increments the sent-email counter in the sheet, pauses for one minute to reduce rate-limit pressure, and returns control to the batch loop for the next client.

### Nodes Involved

- 📊 Update Email Count in Sheet
- ⏱️ Rate Limit Wait (1 Min)

### Node Details

#### 1) 📊 Update Email Count in Sheet

- **Type / role:** `n8n-nodes-base.googleSheets` — increments outreach count for the client.
- **Configuration:** Updates **Database** sheet matching by **Email Address**. Writes:
  - `Email Address`
  - `Number of Emails Sent = (parseInt(existing) || 0) + 1`
- **Key variables / expressions:**
  - `={{ $('📋 Old Client Database').item.json['Email Address'] }}`
  - `={{ (parseInt($('📋 Old Client Database').item.json['Number of Emails Sent']) || 0) + 1 }}`
- **Input connections:** From **📤 Send Re-engagement Email**
- **Output connections:** To **⏱️ Rate Limit Wait (1 Min)**
- **Version-specific notes:** Type version `4.5`.
- **Potential failures / edge cases:**
  - Non-numeric values in `Number of Emails Sent` fall back to `0`.
  - Duplicate email rows can update multiple or unintended records.
  - Sheet update failure may leave the system out of sync with actual sends.

#### 2) ⏱️ Rate Limit Wait (1 Min)

- **Type / role:** `n8n-nodes-base.wait` — pauses before the next loop iteration.
- **Configuration:** Wait unit is `minutes`. No explicit amount is shown, but the node name indicates intended wait time is 1 minute; confirm in the UI.
- **Input connections:** From **📊 Update Email Count in Sheet**
- **Output connections:** Back to **🔄 Loop: Process Each Client**
- **Version-specific notes:** Type version `1.1`.
- **Potential failures / edge cases:**
  - If the duration is not explicitly set, actual wait behavior may differ from the node name.
  - Wait nodes can keep executions open for long periods on large datasets.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Daily Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | 📋 Old Client Database | ## 1. Trigger & Load Clients<br>Schedule trigger fires daily and loads all client records from the "Database" sheet in Google Sheets.<br>Only rows where "Manually Stop Workflow (STOP)" is not triggered are fetched — giving you manual override control directly from the sheet. |
| 📋 Old Client Database | Google Sheets | Loads client records from the Database sheet | ⏰ Daily Schedule Trigger | 🔄 Loop: Process Each Client | ## 1. Trigger & Load Clients<br>Schedule trigger fires daily and loads all client records from the "Database" sheet in Google Sheets.<br>Only rows where "Manually Stop Workflow (STOP)" is not triggered are fetched — giving you manual override control directly from the sheet. |
| 🔄 Loop: Process Each Client | Split In Batches | Processes clients one by one and receives loop-backs | 📋 Old Client Database; 📝 Update: Mark Reply Manually; ⏱️ Rate Limit Wait (1 Min) | 📧 Fetch Client's Latest Email | ## 2. Loop: Process Each Client<br>Splits all loaded clients into individual items and processes them one by one in a loop.<br>- Loop start: picks next client from the list<br>- Loop end: returns here after each client is done<br>- Stops automatically when all clients are processed |
| 📧 Fetch Client's Latest Email | Gmail | Retrieves the most recent email from the client | 🔄 Loop: Process Each Client | 📅 Date Checker (30-Day Window) | ## 3. Check Latest Email from Client<br>Fetches the most recent email received from the client's email address via Gmail.<br>Date Checker then calculates:<br>- Was the email received in the last 30 days?<br>- How many days ago was it?<br>- Formats the date as: DD-Mon-YYYY |
| 📅 Date Checker (30-Day Window) | Code | Computes 30-day recency and formats date fields | 📧 Fetch Client's Latest Email | 🔀 Route: Client Replied Recently? | ## 3. Check Latest Email from Client<br>Fetches the most recent email received from the client's email address via Gmail.<br>Date Checker then calculates:<br>- Was the email received in the last 30 days?<br>- How many days ago was it?<br>- Formats the date as: DD-Mon-YYYY |
| 🔀 Route: Client Replied Recently? | Switch | Branches between manual handling and AI follow-up | 📅 Date Checker (30-Day Window) | 📝 Update: Mark Reply Manually; 📂 Fetch Situation-Based Prompt; 📩 Fetch Followup Message Template; ✅ Update: Mark Running Status | ## 4. Route by Reply Status<br>Checks the result from Date Checker:<br>✅ Replied within 30 days<br>→ Route 0: Mark as "Reply Manually"<br>🔁 No reply in last 30 days<br>→ Route 1: Continue with AI follow-up sequence |
| 📝 Update: Mark Reply Manually | Google Sheets | Marks client for manual reply and logs latest email | 🔀 Route: Client Replied Recently? | 🔄 Loop: Process Each Client | ## 5. Client Replied — Skip<br>Saves latest email and date in sheet.<br>Marks status as "Reply Manually" and skips. |
| ✅ Update: Mark Running Status | Google Sheets | Marks client as actively being processed | 🔀 Route: Client Replied Recently? | 📬 Fetch 10 Email Conversations | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| 📩 Fetch Followup Message Template | Google Sheets | Retrieves follow-up message direction/template | 🔀 Route: Client Replied Recently? | 🔀 Merge: Templates + Conversations | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| 📂 Fetch Situation-Based Prompt | Google Sheets | Retrieves prompt text based on client scenario | 🔀 Route: Client Replied Recently? | 🔀 Merge: Templates + Conversations | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| 📬 Fetch 10 Email Conversations | Gmail | Retrieves recent email history from the client | ✅ Update: Mark Running Status | 📦 Bundle Email Conversations | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| 📦 Bundle Email Conversations | Aggregate | Aggregates conversation texts into one payload | 📬 Fetch 10 Email Conversations | 🔀 Merge: Templates + Conversations | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| 🔀 Merge: Templates + Conversations | Merge | Combines template, prompt, and conversation bundle | 📩 Fetch Followup Message Template; 📂 Fetch Situation-Based Prompt; 📦 Bundle Email Conversations | 📦 Prepare AI Input Bundle | ## 7. Merge & Prepare AI Input<br>Combines template, prompt, and email history into one clean input for the AI agent. |
| 📦 Prepare AI Input Bundle | Aggregate | Aggregates merged fields for AI prompting | 🔀 Merge: Templates + Conversations | 🤖 Re-engagement Email Drafter | ## 7. Merge & Prepare AI Input<br>Combines template, prompt, and email history into one clean input for the AI agent. |
| 🤖 Re-engagement Email Drafter | LangChain Agent | Generates a structured re-engagement email draft | 📦 Prepare AI Input Bundle; 🤖 Claude 3.7 Sonnet LLM; 📋 Email Structure Parser | 📤 Send Re-engagement Email | ## 8. AI Drafts & Sends Email<br>Claude 3.7 writes a personalized re-engagement email using past conversations and client context.<br>Validates structure and sends via Gmail. |
| 🤖 Claude 3.7 Sonnet LLM | Anthropic Chat Model | Provides Claude 3.7 Sonnet as the language model |  | 🤖 Re-engagement Email Drafter | ## 8. AI Drafts & Sends Email<br>Claude 3.7 writes a personalized re-engagement email using past conversations and client context.<br>Validates structure and sends via Gmail. |
| 📋 Email Structure Parser | Structured Output Parser | Enforces structured email output schema |  | 🤖 Re-engagement Email Drafter | ## 8. AI Drafts & Sends Email<br>Claude 3.7 writes a personalized re-engagement email using past conversations and client context.<br>Validates structure and sends via Gmail. |
| 📤 Send Re-engagement Email | Gmail | Sends the AI-generated email via Gmail | 🤖 Re-engagement Email Drafter | 📊 Update Email Count in Sheet | ## 8. AI Drafts & Sends Email<br>Claude 3.7 writes a personalized re-engagement email using past conversations and client context.<br>Validates structure and sends via Gmail. |
| 📊 Update Email Count in Sheet | Google Sheets | Increments the sent-email counter | 📤 Send Re-engagement Email | ⏱️ Rate Limit Wait (1 Min) | ## 9. Update Count & Loop Back<br>Increments email count in sheet.<br>Waits 1 min then loops to next client. |
| ⏱️ Rate Limit Wait (1 Min) | Wait | Pauses execution before processing the next client | 📊 Update Email Count in Sheet | 🔄 Loop: Process Each Client | ## 9. Update Count & Loop Back<br>Increments email count in sheet.<br>Waits 1 min then loops to next client. |
| Sticky Note | Sticky Note | Documentation note on full workflow |  |  | ## AI-Powered Old Client Re-engagement<br>This workflow automatically re-engages old clients by sending personalized AI-written follow-up emails daily — based on past conversations, client industry, and situation-specific templates.<br>It skips clients who have already replied, avoiding spam and keeping outreach professional.<br>## ⚙️ How it works<br>1. Schedule trigger runs the workflow daily<br>2. Loads all old clients from Google Sheets database<br>3. Loops through each client one by one<br>4. Fetches the client's latest email from Gmail<br>5. Checks if client replied in the last 30 days<br>6. If replied → marks "Reply Manually" in sheet → skips to next client<br>7. If no reply → marks "Running" and fetches templates + past conversations<br>8. Merges follow-up template, situation prompt, and email history<br>9. Claude 3.7 Sonnet drafts a personalized re-engagement email<br>10. Email is sent via Gmail automatically<br>11. Sheet updates email count, waits 1 min, then moves to next client<br>## 🛠️ Setup steps<br>☐ Connect Google Sheets OAuth credentials<br>☐ Connect Gmail OAuth credentials<br>☐ Replace YOUR_GOOGLE_SHEET_ID with your actual Sheet ID<br>☐ Replace YOUR_EMAIL@yourdomain.com in Send Email node<br>☐ Add Anthropic API key for Claude 3.7 Sonnet model<br>☐ Fill client data in "Database" sheet<br>☐ Add follow-up templates in "Follow-up Messages" sheet<br>☐ Add situation prompts in "Situation" sheet<br>☐ Activate workflow |
| Sticky Note1 | Sticky Note | Documentation note for trigger/load block |  |  | ## 1. Trigger & Load Clients<br>Schedule trigger fires daily and loads all client records from the "Database" sheet in Google Sheets.<br>Only rows where "Manually Stop Workflow (STOP)" is not triggered are fetched — giving you manual override control directly from the sheet. |
| Sticky Note2 | Sticky Note | Documentation note for loop block |  |  | ## 2. Loop: Process Each Client<br>Splits all loaded clients into individual items and processes them one by one in a loop.<br>- Loop start: picks next client from the list<br>- Loop end: returns here after each client is done<br>- Stops automatically when all clients are processed |
| Sticky Note3 | Sticky Note | Documentation note for latest-email check block |  |  | ## 3. Check Latest Email from Client<br>Fetches the most recent email received from the client's email address via Gmail.<br>Date Checker then calculates:<br>- Was the email received in the last 30 days?<br>- How many days ago was it?<br>- Formats the date as: DD-Mon-YYYY |
| Sticky Note4 | Sticky Note | Documentation note for routing block |  |  | ## 4. Route by Reply Status<br>Checks the result from Date Checker:<br>✅ Replied within 30 days<br>→ Route 0: Mark as "Reply Manually"<br>🔁 No reply in last 30 days<br>→ Route 1: Continue with AI follow-up sequence |
| Sticky Note5 | Sticky Note | Documentation note for manual-reply skip block |  |  | ## 5. Client Replied — Skip<br>Saves latest email and date in sheet.<br>Marks status as "Reply Manually" and skips. |
| Sticky Note6 | Sticky Note | Documentation note for template/context fetch block |  |  | ## 6. Fetch Templates & Conversations<br>Marks client as "Running" in sheet.<br>Fetches follow-up template, situation prompt, and last 10 emails for AI context. |
| Sticky Note7 | Sticky Note | Documentation note for AI input preparation |  |  | ## 7. Merge & Prepare AI Input<br>Combines template, prompt, and email history into one clean input for the AI agent. |
| Sticky Note8 | Sticky Note | Documentation note for AI send block |  |  | ## 8. AI Drafts & Sends Email<br>Claude 3.7 writes a personalized re-engagement email using past conversations and client context.<br>Validates structure and sends via Gmail. |
| Sticky Note9 | Sticky Note | Documentation note for update/loop-back block |  |  | ## 9. Update Count & Loop Back<br>Increments email count in sheet.<br>Waits 1 min then loops to next client. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like **Re-engage old clients with Claude 3.7 Sonnet, Gmail and Google Sheets**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **⏰ Daily Schedule Trigger**
   - Configure it to run daily at your desired time.
   - Confirm your workflow timezone matches your business timezone.

3. **Add a Google Sheets node to load clients**
   - Node type: **Google Sheets**
   - Name: **📋 Old Client Database**
   - Connect it after the trigger.
   - Authenticate with **Google Sheets OAuth2** credentials.
   - Select your spreadsheet ID in place of `YOUR_GOOGLE_SHEET_ID`.
   - Select the **Database** sheet/tab (`gid=0` in the original workflow).
   - Configure it to read rows.
   - Add filtering around the column **Manually Stop Workflow (STOP)** if you want a manual stop mechanism.
   - Ensure your Database sheet includes at least these columns used later:
     - Client Name
     - Email Address
     - Scenario
     - Number of Emails Sent
     - The goal which client wanted to achieve by hiring an agency
     - Detailed Reason Why Contract Got Ended
     - Client is from Industry of
     - Client's Industry Vertical
     - Specific services they used
     - Manually Stop Workflow (STOP)
     - Followup Workflow (Running,Reply Manually)
     - Lastest Email from Client
     - Date of Lastest Email from Client

4. **Add a Split In Batches node**
   - Node type: **Split In Batches**
   - Name: **🔄 Loop: Process Each Client**
   - Connect **📋 Old Client Database** to it.
   - Keep default batch size of 1 so each client is processed individually.

5. **Add a Gmail node to fetch the latest email**
   - Node type: **Gmail**
   - Name: **📧 Fetch Client's Latest Email**
   - Connect it from the second/main iterative output of the Split In Batches node.
   - Authenticate with **Gmail OAuth2** credentials.
   - Operation: **Get Many / getAll**
   - Limit: `1`
   - Set **Simple** to `false`
   - Add sender filter:
     - `={{ $json['Email Address'] }}`

6. **Add a Code node to evaluate reply recency**
   - Node type: **Code**
   - Name: **📅 Date Checker (30-Day Window)**
   - Connect it after the latest-email Gmail node.
   - Paste this logic conceptually:
     - Read the Gmail message date
     - Compute whether it is within the last 30 days
     - Compute `daysDifference`
     - Format a display date like `1-Jan-2025`
   - Recreate the output fields:
     - `isWithinLast30Days`
     - `daysDifference`
     - `thirtyDaysAgo`
     - `formattedDate`
   - Avoid the hardcoded fallback date in production; instead handle missing dates explicitly.

7. **Add a Switch node for routing**
   - Node type: **Switch**
   - Name: **🔀 Route: Client Replied Recently?**
   - Connect it after the Code node.
   - Create two boolean rules:
     - Rule 1: `{{$json.isWithinLast30Days}}` is `true`
     - Rule 2: `{{$json.isWithinLast30Days}}` is `false`

8. **Add a Google Sheets update node for recent replies**
   - Node type: **Google Sheets**
   - Name: **📝 Update: Mark Reply Manually**
   - Connect it to the `true` output of the Switch.
   - Operation: **Update**
   - Spreadsheet: same Database sheet
   - Match by column: **Email Address**
   - Map fields:
     - `Email Address` = `{{ $('📋 Old Client Database').item.json['Email Address'] }}`
     - `Lastest Email from Client` = `{{ $('📧 Fetch Client\'s Latest Email').item.json.text }}`
     - `Date of Lastest Email from Client` = `{{ $('📅 Date Checker (30-Day Window)').item.json.formattedDate }}`
     - `Followup Workflow (Running,Reply Manually)` = `Reply Manually`
   - Connect this node back to **🔄 Loop: Process Each Client** to continue with the next client.

9. **Add a Google Sheets update node for running status**
   - Node type: **Google Sheets**
   - Name: **✅ Update: Mark Running Status**
   - Connect it to the `false` output of the Switch.
   - Operation: **Update**
   - Match by **Email Address**
   - Map:
     - `Email Address` = current client email
     - `Followup Workflow (Running,Reply Manually)` = `Running`
     - Optionally clear latest email/date fields; in the original workflow they are set to `=`, but empty strings are safer in many cases.

10. **Add a Google Sheets node for follow-up templates**
    - Node type: **Google Sheets**
    - Name: **📩 Fetch Followup Message Template**
    - Connect it to the `false` output of the Switch.
    - Set **On Error** to continue, if you want the same fault-tolerant behavior.
    - Read from the **Follow-up Messages** tab.
    - Filter by:
      - `Scenario = {{ $('📋 Old Client Database').item.json.Scenario }}`
      - `Number of Emails Sent = {{ $('📋 Old Client Database').item.json['Number of Emails Sent'] }}`
    - Ensure the tab contains at least:
      - Scenario
      - Number of Emails Sent
      - Followup Message Direction

11. **Add a Google Sheets node for situation prompts**
    - Node type: **Google Sheets**
    - Name: **📂 Fetch Situation-Based Prompt**
    - Connect it to the `false` output of the Switch.
    - Set **On Error** to continue if desired.
    - Read from the **Situation** tab.
    - Filter by:
      - `Scenario = {{ $('📋 Old Client Database').item.json.Scenario }}`
    - Ensure the tab contains at least:
      - Scenario
      - Prompt

12. **Add a Gmail node to fetch conversation history**
    - Node type: **Gmail**
    - Name: **📬 Fetch 10 Email Conversations**
    - Connect it after **✅ Update: Mark Running Status**
    - Operation: **Get Many / getAll**
    - Limit: `10`
    - Set **Simple** to `false`
    - Filter sender:
      - `={{ $json['Email Address'] }}`

13. **Add an Aggregate node to bundle email texts**
    - Node type: **Aggregate**
    - Name: **📦 Bundle Email Conversations**
    - Connect it after **📬 Fetch 10 Email Conversations**
    - Aggregate field:
      - `text`

14. **Add a Merge node**
    - Node type: **Merge**
    - Name: **🔀 Merge: Templates + Conversations**
    - Set number of inputs to `3`
    - Connect:
      - Input 1: **📩 Fetch Followup Message Template**
      - Input 2: **📂 Fetch Situation-Based Prompt**
      - Input 3: **📦 Bundle Email Conversations**

15. **Add a second Aggregate node to prepare the AI payload**
    - Node type: **Aggregate**
    - Name: **📦 Prepare AI Input Bundle**
    - Connect it after the Merge node.
    - Aggregate these fields:
      - `Prompt`
      - `text`
      - `Followup Message Direction`

16. **Add an Anthropic Chat Model node**
    - Node type: **Anthropic Chat Model**
    - Name: **🤖 Claude 3.7 Sonnet LLM**
    - Create/connect **Anthropic API credentials**
    - Select model:
      - `claude-3-7-sonnet-20250219`

17. **Add a Structured Output Parser node**
    - Node type: **Structured Output Parser**
    - Name: **📋 Email Structure Parser**
    - Define a manual JSON schema with:
      - `subject` string
      - `body.greeting` string
      - `body.content` string
      - `body.closing` string
    - Make `subject`, `body`, and `body.content` required

18. **Add an AI Agent node**
    - Node type: **AI Agent**
    - Name: **🤖 Re-engagement Email Drafter**
    - Connect main input from **📦 Prepare AI Input Bundle**
    - Connect the Anthropic model node to the agent’s **language model** input
    - Connect the parser node to the agent’s **output parser** input
    - Set prompt type to **Define below**
    - Add a system message such as:
      - `Draft an email based on the recent conversation done with the client convincing him that, we have now improved and help him achieve his goals.`
    - Build the user prompt using:
      - Prompt from the Situation sheet
      - Follow-up direction from the Follow-up Messages sheet
      - Bundled email history
      - Client metadata from the Database sheet:
        - Email address
        - Goal
        - End reason
        - Industry
        - Industry vertical
        - Services used

19. **Add a Gmail send node**
    - Node type: **Gmail**
    - Name: **📤 Send Re-engagement Email**
    - Connect it after the AI Agent.
    - Operation: **Send**
    - **Important:** replace the placeholder recipient.
      - Preferred production value: `={{ $('📋 Old Client Database').item.json['Email Address'] }}`
    - Subject:
      - `={{ $json.output.subject }}`
    - Body:
      - `={{ $json.output.body.greeting }}\n\n{{ $json.output.body.content }}`
    - Email type: **Text**
    - Disable attribution if desired.
    - If you want the closing included, append:
      - `\n\n{{ $json.output.body.closing }}`

20. **Add a Google Sheets update node for the email count**
    - Node type: **Google Sheets**
    - Name: **📊 Update Email Count in Sheet**
    - Connect it after the Gmail send node.
    - Operation: **Update**
    - Match by **Email Address**
    - Update:
      - `Email Address = {{ $('📋 Old Client Database').item.json['Email Address'] }}`
      - `Number of Emails Sent = {{ (parseInt($('📋 Old Client Database').item.json['Number of Emails Sent']) || 0) + 1 }}`

21. **Add a Wait node**
    - Node type: **Wait**
    - Name: **⏱️ Rate Limit Wait (1 Min)**
    - Connect it after the email-count update node.
    - Configure wait duration to **1 minute**.

22. **Close the loop**
    - Connect **⏱️ Rate Limit Wait (1 Min)** back to **🔄 Loop: Process Each Client**.
    - This ensures the next client starts only after the pause.

23. **Validate column names carefully**
    - The workflow relies on exact Google Sheets headers, including typos like:
      - `Lastest Email from Client`
      - `Date of Lastest Email from Client`
    - If you rename them in the sheet, update all node mappings and expressions accordingly.

24. **Configure credentials**
    - **Google Sheets OAuth2** for all Sheets nodes
    - **Gmail OAuth2** for both Gmail read and send nodes
    - **Anthropic API** for Claude 3.7 Sonnet

25. **Prepare the spreadsheet tabs**
    - **Database** tab: client master records
    - **Follow-up Messages** tab: scenario + email-count to message-direction mapping
    - **Situation** tab: scenario to prompt mapping

26. **Test with one client first**
    - Add one test row in the Database sheet.
    - Confirm Gmail search returns expected messages.
    - Confirm prompt/template rows exist for the row’s Scenario and Number of Emails Sent.
    - Confirm the send node goes to your test address before enabling real outreach.

27. **Activate the workflow**
    - After validating the full path, turn on the workflow so the schedule trigger runs automatically.

### Suggested production improvements while rebuilding

28. **Fix the send recipient**
   - The current exported workflow sends to `user@example.com`.
   - Replace it with the actual client email expression.

29. **Handle clients with no prior emails**
   - Add an IF or fallback branch after Gmail fetch nodes so clients with no history can still be processed safely.

30. **Replace the hardcoded fallback date**
   - In the Code node, do not default missing dates to a static historical value.
   - Instead, explicitly mark such clients as having no recent reply.

31. **Clarify the stop filter**
   - Make the Google Sheets filter explicit, for example:
     - only process rows where `Manually Stop Workflow (STOP)` is empty or not equal to `STOP`.

32. **Add failure notifications**
   - Optionally add Slack, email, or another logging path when template lookup, prompt lookup, or AI generation fails.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI-Powered Old Client Re-engagement: automatically sends personalized AI-written follow-up emails daily based on past conversations, client industry, and situation-specific templates. Skips clients who have already replied to avoid spam and keep outreach professional. | Workflow-wide concept |
| Setup checklist from the workflow note: connect Google Sheets OAuth credentials, connect Gmail OAuth credentials, replace `YOUR_GOOGLE_SHEET_ID`, replace the placeholder send address, add Anthropic API key, fill the Database sheet, add Follow-up Messages sheet entries, add Situation sheet entries, then activate the workflow. | Workflow-wide setup guidance |
| The exported workflow contains placeholder values such as `YOUR_GOOGLE_SHEET_ID` and `user@example.com`; these must be replaced before production use. | Configuration warning |
| The workflow uses Claude 3.7 Sonnet via Anthropic model `claude-3-7-sonnet-20250219`. | AI model dependency |
| Several fields and headers are referenced by exact literal names, including apostrophes and typos. Keep column names consistent or update all expressions. | Google Sheets schema dependency |