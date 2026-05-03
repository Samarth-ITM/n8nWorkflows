Triage LinkedIn notifications and InMails using Gmail, OpenAI, Notion, and Slack

https://n8nworkflows.xyz/workflows/triage-linkedin-notifications-and-inmails-using-gmail--openai--notion--and-slack-13121


# Triage LinkedIn notifications and InMails using Gmail, OpenAI, Notion, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Triage LinkedIn notifications and InMails using Gmail, OpenAI, Notion, and Slack  
**Workflow name (in JSON):** LinkedIn Triage with Labels  
**Purpose:** Runs daily, pulls the last 24h of LinkedIn-related emails from Gmail (via a label), filters out common automated noise, fetches full message bodies, uses an AI agent to classify/summarize and propose reply drafts, stores results in Notion, and sends a Slack DM only for items needing a quick reply.

### 1.1 Trigger & Gmail Intake (last 24h)
- Scheduled daily trigger
- Fetch up to 500 labeled Gmail messages received since start of previous day

### 1.2 Noise Filtering (keep real messages/InMails)
- Two sequential filters:
  - Keep only messages whose sender/From contains “invitations” or “messages”
  - Exclude typical automated LinkedIn notifications by snippet/subject phrases

### 1.3 Full Email Retrieval
- For each kept item, fetch full email content (not just snippet)

### 1.4 AI Triage with Structured Output
- Send message metadata + truncated body excerpt to an AI agent
- Enforce a JSON schema using a structured output parser

### 1.5 Persist & Notify
- Store triage record in Notion database
- If action is `reply_quick`, DM the user on Slack with summary and draft

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Gmail Intake
**Overview:** Starts the workflow daily and retrieves LinkedIn-labeled emails from the past day.  
**Nodes involved:** `Daily Trigger`, `Pull Messages From with LinkedIn tags`

#### Node: Daily Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – workflow entry point.
- **Config (interpreted):** Runs on a daily interval (default schedule rule).
- **Outputs:** Triggers the Gmail “getAll” node.
- **Edge cases / failures:**
  - Misconfigured schedule/timezone expectations (runs according to n8n instance timezone).
  - No data emitted if schedule is disabled (workflow inactive).

#### Node: Pull Messages From with LinkedIn tags
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) – list messages.
- **Operation:** `getAll`
- **Key configuration:**
  - `limit: 500`
  - **Label filter:** `labelIds: ["REPLACE_WITH_YOUR_GMAIL_LABEL_ID"]`
  - **Date filter:** `receivedAfter = {{ $now.minus({ days: 1 }).startOf('day').toISO() }}`
    - Pulls messages received after the start of “yesterday” (can be close to ~24–48h window depending on run time).
- **Input:** From `Daily Trigger`.
- **Output:** Array of message metadata items, each with fields like `id`, `From`, `Subject`, `snippet`, etc.
- **Edge cases / failures:**
  - Gmail auth/OAuth issues or insufficient scopes.
  - Wrong label ID → returns nothing.
  - Large volumes > 500 will be truncated; older items may be missed.

---

### Block 2 — Noise Filtering (LinkedIn messages only)
**Overview:** Removes non-message notifications and common automated noise before spending tokens/cost on AI.  
**Nodes involved:** `First Filter for non direct messages or invites`, `Second Filter for non direct messages or invites`

#### Node: First Filter for non direct messages or invites
- **Type / role:** Filter (`n8n-nodes-base.filter`) – keep only likely message/invite senders.
- **Logic (OR):**
  - `{{ $json.From }}` **contains** `"invitations"` OR
  - `{{ $json.From }}` **contains** `"messages"`
- **Input:** Gmail list output.
- **Output:** Only items that match.
- **Edge cases / failures:**
  - LinkedIn sender formats differ by locale; `From` might not contain these substrings → false negatives.
  - Some real InMails may come from different “From” strings.

#### Node: Second Filter for non direct messages or invites
- **Type / role:** Filter – exclude known automated content.
- **Logic (AND of NOT contains):**
  - `snippet` not contains `"You're getting noticed"`
  - `Subject` not contains `"Welcome to"`
  - `snippet` not contains `"LinkedIn Page admin"`
  - `snippet` not contains `"company verification"`
- **Input:** Output of first filter.
- **Output:** Items likely to be real conversations/InMails.
- **Edge cases / failures:**
  - Language variations cause misses; update phrases to match your LinkedIn email language.
  - If `snippet` or `Subject` is missing/unexpected type, strict validation may cause filter evaluation issues.

---

### Block 3 — Fetch Full Email Body
**Overview:** Retrieves the full message details (including body) so the AI agent can classify content reliably.  
**Nodes involved:** `Pull body of messages`

#### Node: Pull body of messages
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) – fetch a single message by ID.
- **Operation:** `get`
- **Key configuration:**
  - `messageId = {{ $json.id }}`
  - `simple: false` (requests more complete/structured payload)
- **Input:** Filtered message metadata.
- **Output:** Full message object; workflow later uses:
  - `subject`, `date`, `from.value[0].name`, and `text` (body text).
- **Edge cases / failures:**
  - Message ID missing/undefined if upstream node schema differs.
  - Gmail API can return bodies in HTML only; `text` might be empty depending on parsing.

---

### Block 4 — AI Triage with Structured Output
**Overview:** Sends email details to an AI agent and forces a stable JSON response format for downstream mapping.  
**Nodes involved:** `Open AI Chat Model`, `Structured Output Parser`, `LinkedIn Triage Agent`, `OpenAI Chat Model` (unused)

#### Node: Open AI Chat Model
- **Type / role:** OpenAI Chat Model for LangChain (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) – the LLM used by the agent.
- **Model:** `gpt-5.2`
- **Options:** `maxTokens: 6000`
- **Connections:** Provides `ai_languageModel` input to `LinkedIn Triage Agent`.
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - High `maxTokens` increases cost and may hit account limits.
  - Long inputs can still exceed context limits; body is truncated upstream in the agent prompt.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) – validates/coerces model output into a fixed JSON schema.
- **Behavior:**
  - `schemaType: manual` with an explicit JSON Schema
  - `autoFix: true` (attempts to repair near-valid JSON)
  - `onError: continueRegularOutput` (does not hard-fail the workflow on parse error)
  - `retryOnFail: true`
- **Schema highlights (required):**
  - `message_id` (string)
  - `subject` (string)
  - `action` enum: `reply_quick | review | ignore | block`
  - `relevancy_score` (0–100 integer)
  - `sales_likelihood` (0–1 number)
  - `summary` (≤ 220 chars)
  - Optional: `date_iso`, `from_name`, `reply_draft` (≤ 700 chars), `tags` (array up to 6)
- **Connections:** Supplies `ai_outputParser` to the agent.
- **Edge cases / failures:**
  - If the agent returns text outside JSON, autoFix may still fail; because errors continue, downstream nodes might see missing `output.*` fields.
  - Strict `additionalProperties: false` can reject extra keys produced by the model.

#### Node: LinkedIn Triage Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) – orchestrates prompt + model + parser.
- **Prompt payload (built from Gmail message):**
  - Includes a hard marker: `MessageID(DO NOT CHANGE):{{ $json.id }}`
  - Subject, date, sender name, and body excerpt: `{{ ($json.text || '').slice(0, 4000) }}`
- **System message:** Instructs aggressive sales/spam filtering, suggests replies for real messages, and enforces “Return ONLY valid JSON matching the schema.” Includes fallback guidance if unsure.
- **Connections:**
  - Main input: from `Pull body of messages`
  - `ai_languageModel`: from `Open AI Chat Model`
  - `ai_outputParser`: from `Structured Output Parser`
  - Main output: to `Filter out Irrelevant and Sales heavy requests`
- **Error handling:** `onError: continueRegularOutput`, `retryOnFail: true`
- **Edge cases / failures:**
  - Sender name path `from.value[0].name` may not exist for some emails → expression resolves to empty/throws depending on n8n evaluation.
  - If parser fails, `output` may be absent; downstream filters referencing `$json.output.sales_likelihood` can error or filter out everything.
  - Truncation to 4000 chars may omit key context for classification.

#### Node: OpenAI Chat Model (unused)
- **Type / role:** Another OpenAI chat model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with `gpt-5.2`.
- **Connections:** Only connected to `Structured Output Parser` as `ai_languageModel`, but the parser does not require a language model in typical setups; this node is not feeding the agent.
- **Impact:** Likely redundant; can be removed unless your n8n version requires it for parser-assisted fixing (verify in your instance).
- **Edge cases:** Confusion about which model is actually used; ensure the agent is connected to the intended model node.

---

### Block 5 — Filter, Store in Notion, and Slack Notify
**Overview:** Keeps only relevant/non-sales items, stores them in Notion, and sends Slack DM for `reply_quick`.  
**Nodes involved:** `Filter out Irrelevant and Sales heavy requests`, `Put into Ticketing System`, `Filter out reply quick tickets`, `Send Message to myself`

#### Node: Filter out Irrelevant and Sales heavy requests
- **Type / role:** Filter – gate Notion writes and Slack notifications.
- **Conditions (AND):**
  - `{{ $json.output.sales_likelihood }} < 0.8`
  - `{{ $json.output.relevancy_score }} > 40`
- **Input:** Agent output (expects `output.*`).
- **Output:** Only “worth attention” items.
- **Edge cases / failures:**
  - If `output` missing (parser failed), comparisons may fail or evaluate unexpectedly.
  - Threshold tuning: aggressive thresholds may drop genuine leads.

#### Node: Put into Ticketing System
- **Type / role:** Notion (`n8n-nodes-base.notion`) – create database page.
- **Resource:** `databasePage` (create a new page in a database)
- **Database:** `REPLACE_WITH_YOUR_NOTION_DATABASE_ID`
- **Mapped properties (must exist in Notion DB with matching types):**
  - `date_iso` (rich text) ← `{{ $json.output.date_iso }}`
  - `from_name` (rich text) ← `{{ $json.output.from_name }}`
  - `message_id` (title) ← `{{ $json.output.message_id }}`
  - `relevancy_score` (number) ← `{{ $json.output.relevancy_score }}`
  - `reply_draft` (rich text) ← `{{ $json.output.reply_draft || "" }}`
  - `sales_likelihood` (number) ← `{{ $json.output.sales_likelihood }}`
  - `subject` (rich text) ← `{{ $json.output.subject }}`
  - `summary` (rich text) ← `{{ $json.output.summary }}`
  - `tags` (multi-select) ← `{{ $json.output.action }}`
    - Note: This maps `action` into a multi-select; ensure the multi-select options include the possible actions or Notion will create new options dynamically (depending on permissions/API behavior).
- **Input:** Filtered agent output.
- **Output:** Notion page object with `properties.*` used later by Slack filter/message.
- **Edge cases / failures:**
  - Notion auth/permissions; database not shared with integration.
  - Property name/type mismatch causes runtime errors.
  - If `date_iso/from_name` are not returned by the model (they are optional in schema), Notion fields may be empty.

#### Node: Filter out reply quick tickets
- **Type / role:** Filter – only notify Slack for quick replies.
- **Condition:** `{{ $json.properties.tags.multi_select[0].name }} == "reply_quick"`
  - Relies on Notion’s returned page structure and the first multi-select value.
- **Input:** Notion page creation output.
- **Output:** Only pages tagged `reply_quick`.
- **Edge cases / failures:**
  - If `tags` is empty, `multi_select[0]` is undefined → expression may fail.
  - If multiple select values exist, relying on index `[0]` may miss `reply_quick` when not first.

#### Node: Send Message to myself
- **Type / role:** Slack (`n8n-nodes-base.slack`) – send a direct message.
- **Authentication:** OAuth2
- **Target:** `select: user`, user id `REPLACE_WITH_YOUR_SLACK_USER_ID`
- **Message text:**
  - Uses Notion page properties:
    - From: `from_name.rich_text[0].text.content`
    - Summary: `summary.rich_text[0].text.content`
    - Draft: `reply_draft.rich_text[0].text.content`
- **Input:** Filtered Notion pages.
- **Edge cases / failures:**
  - Slack OAuth scopes missing (e.g., DM permission).
  - Notion rich_text arrays can be empty → indexing `[0]` can fail.
  - Draft may be empty; expression should guard but currently does not.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger | scheduleTrigger | Daily workflow entry point | — | Pull Messages From with LinkedIn tags | LinkedIn notifications and InMails triage (runs daily; Gmail label; AI classify; store Notion; Slack for reply_quick; setup steps) |
| Pull Messages From with LinkedIn tags | gmail | List labeled Gmail messages (last day) | Daily Trigger | First Filter for non direct messages or invites | Gmail intake and noise filters (Pull messages from label; keep real messages/InMails; remove automated; update phrases per language) |
| First Filter for non direct messages or invites | filter | Keep only LinkedIn messages/invitations senders | Pull Messages From with LinkedIn tags | Second Filter for non direct messages or invites | Gmail intake and noise filters (Pull messages from label; keep real messages/InMails; remove automated; update phrases per language) |
| Second Filter for non direct messages or invites | filter | Exclude common automated LinkedIn notifications | First Filter for non direct messages or invites | Pull body of messages | Gmail intake and noise filters (Pull messages from label; keep real messages/InMails; remove automated; update phrases per language) |
| Pull body of messages | gmail | Fetch full email body by message id | Second Filter for non direct messages or invites | LinkedIn Triage Agent | Gmail intake and noise filters (Pull messages from label; keep real messages/InMails; remove automated; update phrases per language) |
| Open AI Chat Model | lmChatOpenAi | LLM used by agent (high token cap) | — (AI connection) | LinkedIn Triage Agent (ai_languageModel) | AI triage (sends subject/sender/date/body excerpt; returns structured fields) |
| Structured Output Parser | outputParserStructured | Enforce/repair schema-valid JSON output | OpenAI Chat Model (ai_languageModel) | LinkedIn Triage Agent (ai_outputParser) | AI triage (sends subject/sender/date/body excerpt; returns structured fields) / Privacy note (sends email to LLM; reduce/redact/store summaries if needed) / Cost and reliability (maxTokens high; parsing failures; consider fallback) |
| OpenAI Chat Model | lmChatOpenAi | Redundant/unused model node (connected to parser) | — | Structured Output Parser (ai_languageModel) | AI triage (sends subject/sender/date/body excerpt; returns structured fields) |
| LinkedIn Triage Agent | agent | Classify, score, summarize, propose reply drafts | Pull body of messages | Filter out Irrelevant and Sales heavy requests | AI triage (sends subject/sender/date/body excerpt; returns structured fields) / Privacy note (sends email to LLM; reduce/redact/store summaries if needed) / Cost and reliability (maxTokens high; parsing failures; consider fallback) |
| Filter out Irrelevant and Sales heavy requests | filter | Keep relevant, low-sales items | LinkedIn Triage Agent | Put into Ticketing System | Outputs (Notion stores all triaged; Slack only for reply_quick) |
| Put into Ticketing System | notion | Create Notion database page for triage record | Filter out Irrelevant and Sales heavy requests | Filter out reply quick tickets | Outputs (Notion stores all triaged; Slack only for reply_quick) |
| Filter out reply quick tickets | filter | Only notify for reply_quick | Put into Ticketing System | Send Message to myself | Outputs (Notion stores all triaged; Slack only for reply_quick) |
| Send Message to myself | slack | Slack DM with summary and reply draft | Filter out reply quick tickets | — | Outputs (Notion stores all triaged; Slack only for reply_quick) |

Sticky notes present but not listed as rows:  
- Sticky Note: LinkedIn notifications and InMails triage  
- Sticky Note2: Gmail intake and noise filters  
- Sticky Note4: AI triage  
- Sticky Note5: Privacy note  
- Sticky Note9: Outputs  
- Sticky Note10: Cost and reliability

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **LinkedIn Triage with Labels** (or your choice).
   - Keep it inactive until credentials and IDs are set.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: Daily (choose desired time; confirm timezone).

3. **Add Gmail node to list messages**
   - Node: **Gmail** → Operation: **Get Many / getAll**
   - Set **Limit** to `500`
   - Filters:
     - **Label IDs:** set to your LinkedIn label ID (create a Gmail label and obtain its ID)
     - **Received After:** expression  
       `{{$now.minus({ days: 1 }).startOf('day').toISO()}}`
   - Credentials: connect Gmail OAuth with scopes allowing message read.

4. **Connect:** Schedule Trigger → Gmail (getAll)

5. **Add Filter #1 (keep invites/messages)**
   - Node: **Filter**
   - Condition group: **OR**
     - `{{$json.From}}` contains `invitations`
     - `{{$json.From}}` contains `messages`
   - Connect: Gmail (getAll) → Filter #1

6. **Add Filter #2 (exclude automated noise)**
   - Node: **Filter**
   - Condition group: **AND**
     - `{{$json.snippet}}` not contains `You're getting noticed`
     - `{{$json.Subject}}` not contains `Welcome to`
     - `{{$json.snippet}}` not contains `LinkedIn Page admin`
     - `{{$json.snippet}}` not contains `company verification`
   - Connect: Filter #1 → Filter #2

7. **Add Gmail node to fetch full message**
   - Node: **Gmail** → Operation: **Get (by ID)**
   - Message ID: `{{$json.id}}`
   - Set **Simple** to **false** (for richer payload)
   - Connect: Filter #2 → Gmail (get)

8. **Add OpenAI Chat Model (the one used by the agent)**
   - Node: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-5.2` (or available equivalent in your account)
   - Options: set `maxTokens` to `6000` (or lower for cost control)
   - Credentials: OpenAI API key / compatible credential in n8n.

9. **Add Structured Output Parser**
   - Node: **Structured Output Parser**
   - Schema type: **Manual**
   - Paste the JSON Schema (same required fields and enums as in the workflow)
   - Enable **Auto-fix**
   - Set node error handling: **Continue on error** (optional but matches this workflow)

10. **Add AI Agent**
    - Node: **AI Agent (LangChain Agent)**
    - Prompt type: “Define” (custom)
    - Text (prompt body), using expressions:
      ```
      MessageID(DO NOT CHANGE):{{$json.id}}
      Subject: {{$json.subject}}
      Date: {{$json.date}}
      Sender: {{$json.from.value[0].name}}
      Body: {{($json.text || '').slice(0, 4000)}}
      ```
    - System message: set the triage rules (sales/spam aggressive, actions, JSON-only).
    - Enable output parsing and attach the **Structured Output Parser** to the agent.
    - Connect the agent’s language model input to **Open AI Chat Model**.

11. **Connect AI chain**
    - Gmail (get full message) → AI Agent (main)
    - Open AI Chat Model → AI Agent (ai_languageModel)
    - Structured Output Parser → AI Agent (ai_outputParser)

12. **Add Filter for relevance/sales**
    - Node: **Filter**
    - Conditions (AND):
      - `{{$json.output.sales_likelihood}}` < `0.8`
      - `{{$json.output.relevancy_score}}` > `40`
    - Connect: AI Agent → this Filter

13. **Add Notion node to create a database page**
    - Node: **Notion** → Resource: **Database Page** → Operation: **Create**
    - Set Database ID to your database.
    - Create matching Notion properties (names and types must match your mappings), then map:
      - `message_id` (Title) ← `{{$json.output.message_id}}`
      - `subject` (Rich text) ← `{{$json.output.subject}}`
      - `summary` (Rich text) ← `{{$json.output.summary}}`
      - `reply_draft` (Rich text) ← `{{$json.output.reply_draft || ""}}`
      - `from_name` (Rich text) ← `{{$json.output.from_name}}`
      - `date_iso` (Rich text) ← `{{$json.output.date_iso}}`
      - `relevancy_score` (Number) ← `{{$json.output.relevancy_score}}`
      - `sales_likelihood` (Number) ← `{{$json.output.sales_likelihood}}`
      - `tags` (Multi-select) ← `{{$json.output.action}}`
    - Credentials: Notion integration token; share the database with the integration.
    - Connect: relevance/sales Filter → Notion

14. **Add Filter for reply_quick**
    - Node: **Filter**
    - Condition:
      - `{{$json.properties.tags.multi_select[0].name}}` equals `reply_quick`
    - Connect: Notion → reply_quick Filter

15. **Add Slack node (DM)**
    - Node: **Slack** → Operation: “Send message” (direct message to user)
    - Authentication: OAuth2
    - Recipient: select **User** and set your user ID (or change to channel)
    - Text (using Notion output):
      ```
      Daily LinkedIn:
      Subject: {{$json.properties.from_name.rich_text[0].text.content}}
      Summary: {{$json.properties.summary.rich_text[0].text.content}}

      Possible drafts:
      {{$json.properties.reply_draft.rich_text[0].text.content}}
      ```
    - Connect: reply_quick Filter → Slack

16. **Test run**
    - Run once manually.
    - Verify:
      - Gmail label ID works
      - Filters aren’t excluding everything
      - AI output contains `output.*` fields
      - Notion property types match
      - Slack message renders (no missing rich_text indexes)

17. **Activate workflow**
    - Turn workflow active after validating a few days of results.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow runs daily and turns LinkedIn email noise into a short list of messages worth your attention… Save the result to Notion… Optionally notify you in Slack for items marked reply_quick.” | Sticky note: LinkedIn notifications and InMails triage |
| Setup steps: create Gmail label + label id; connect Gmail/OpenAI/Notion/Slack; replace Notion DB id and ensure properties; replace Slack user id; run once then activate | Sticky note: LinkedIn notifications and InMails triage |
| Tip: update filter phrases to match your LinkedIn email language | Sticky note: Gmail intake and noise filters |
| Privacy note: sends email content to an LLM; reduce body length, redact, or store only summaries if needed | Sticky note: Privacy note |
| Cost/reliability: `maxTokens` is high; if parsing fails reduce body length or simplify schema; consider fallback action=review | Sticky note: Cost and reliability |