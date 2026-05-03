Qualify and route inbound leads with Claude, Gmail, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-and-route-inbound-leads-with-claude--gmail--slack-and-google-sheets-14836


# Qualify and route inbound leads with Claude, Gmail, Slack and Google Sheets

Now I have all the information I need. Let me create the comprehensive reference document. 1. Workflow Overview

This workflow implements an **AI Lead Qualification Agent** that automatically captures inbound form submissions, uses Anthropic's Claude Sonnet to score each lead from 1 to 10 against a configurable ideal customer profile, and routes the lead into one of three tiers (Hot, Warm, Cold). Each tier triggers tier-specific actions: a personalised email reply, an optional Slack team notification (Hot only), and a row appended to Google Sheets for record-keeping.

**Target use cases:** B2B sales teams, SaaS companies, or any organisation that receives inbound enquiries and wants to prioritise high-fit leads automatically while ensuring every prospect receives a timely response.

**Logical blocks:**

| Block | # | Purpose |
|---|---|---|
| 1.1 Input Reception | Inbound Lead Form, Extract Lead Fields | Capture the lead via an embedded n8n form and normalise field names for downstream use. |
| 1.2 AI Scoring | Score Lead Intent, Claude Sonnet, Parse Score | Send the lead's data to Claude, receive a JSON score/tier/reasoning, and parse it into structured fields. |
| 1.3 Score-Based Routing | Route by Score | Split the flow into Hot (7–10), Warm (4–6), or Cold (1–3) branches using a Switch node. |
| 1.4 Hot Lead Branch | Notify Team – Hot Lead, Hot Lead Reply, Log to Sheets – Hot | Alert the team on Slack, send an enthusiastic personalised email with a calendar link, and log the lead. |
| 1.5 Warm Lead Branch | Warm Lead Reply, Log to Sheets – Warm | Send a friendly exploratory email requesting more detail, and log the lead. |
| 1.6 Cold Lead Branch | Cold Lead Reply, Log to Sheets – Cold | Send a respectful decline email, and log the lead for records. |

---

### 2. Block-by-Block Analysis

#### Block 1.1 — Input Reception

**Overview:** An n8n-native form collects the lead's name, work email, company name, company size (dropdown), and message. The form trigger generates a public URL on activation. The Set node immediately renames and normalises these fields into consistent variable names used everywhere downstream.

**Nodes Involved:** Inbound Lead Form, Extract Lead Fields

---

**Node: Inbound Lead Form**

- **Type:** `n8n-nodes-base.formTrigger` (v2.2)
- **Technical role:** Webhook-based entry point. Generates a unique form URL when the workflow is active. Submissions trigger execution.
- **Configuration:**
  - Form title: "Get in Touch"
  - Form description: "Tell us a bit about yourself and what you are looking for."
  - Response mode: `lastNode` (the last node executed determines what the submitter sees on the confirmation page).
  - Fields (all required):
    1. **Full Name** — text input
    2. **Work Email** — email input
    3. **Company Name** — text input
    4. **Company Size** — dropdown: `1-10`, `11-50`, `51-200`, `201-500`, `500+`
    5. **How can we help you?** — textarea
- **Key expressions/variables:** None at this node; raw form field names are passed as-is.
- **Input connections:** None (entry point).
- **Output connections:** → Extract Lead Fields (main, index 0)
- **Version-specific requirements:** formTrigger v2.2 is only available in n8n ≥ 1.x with the updated form builder.
- **Edge cases / potential failures:**
  - If the workflow is deactivated, the form URL returns 404.
  - Webhook path collisions if the webhook ID is reused.
  - Field validation is enforced client-side; malformed email strings may still pass and cause downstream Gmail send failures.

---

**Node: Extract Lead Fields**

- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical role:** Field mapping / normalisation. Renames raw form fields into canonical variables and adds an ISO timestamp.
- **Configuration — Assignments:**

| Output Field | Type | Expression |
|---|---|---|
| `lead_name` | string | `{{ $json['Full Name'] }}` |
| `lead_email` | string | `{{ $json['Work Email'] }}` |
| `lead_company` | string | `{{ $json['Company Name'] }}` |
| `company_size` | string | `{{ $json['Company Size'] }}` |
| `lead_message` | string | `{{ $json['How can we help you?'] }}` |
| `timestamp` | string | `{{ new Date().toISOString() }}` |

- **Key expressions/variables:** `$json['Full Name']`, `$json['Work Email']`, etc. from the form; `new Date().toISOString()` for the timestamp.
- **Input connections:** ← Inbound Lead Form (main, index 0)
- **Output connections:** → Score Lead Intent (main, index 0)
- **Edge cases / potential failures:**
  - If any required form field is empty (shouldn't happen with required validation), downstream expressions resolve to empty strings.
  - The timestamp is generated at execution time, not at submission time; if there is queue delay, the timestamp may be slightly later than the actual submission moment.

---

#### Block 1.2 — AI Scoring

**Overview:** The lead data is sent to Claude Sonnet via a LangChain chain node. Claude reads the lead details against a defined ideal customer profile and returns a JSON object with `score`, `tier`, and `reasoning`. A Code node then parses Claude's text output, extracts the JSON, and merges it back with the original lead fields so everything flows forward as a single item.

**Nodes Involved:** Score Lead Intent, Claude Sonnet, Parse Score

---

**Node: Score Lead Intent**

- **Type:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.4)
- **Technical role:** LangChain LLM chain that combines a system-level prompt with lead data, sends it to the connected language model (Claude Sonnet), and returns the model's text response.
- **Configuration:**
  - Prompt type: `define` (inline prompt defined in the node).
  - Prompt text (key excerpt):
    - Role: "You are a B2B lead qualification specialist."
    - Ideal Customer Profile:
      - Small to mid-size businesses (1–200 employees)
      - Looking for AI/workflow/sales automation
      - Has a clear problem or goal mentioned
      - Decision maker or senior role
    - Lead details injected via expressions: `{{ $json.lead_name }}`, `{{ $json.lead_company }}`, `{{ $json.company_size }}`, `{{ $json.lead_message }}`
    - Output constraint: Respond ONLY with valid JSON: `{"score": 8, "tier": "Hot", "reasoning": "One sentence explaining the score"}`
    - Score guide: 7–10 = Hot, 4–6 = Warm, 1–3 = Cold
- **Key expressions/variables:** `$json.lead_name`, `$json.lead_company`, `$json.company_size`, `$json.lead_message`
- **Input connections:**
  - Main input: ← Extract Lead Fields (main, index 0)
  - AI language model input: ← Claude Sonnet (ai_languageModel, index 0)
- **Output connections:** → Parse Score (main, index 0)
- **Version-specific requirements:** chainLlm v1.4 requires the n8n LangChain nodes package. Available from n8n ≥ 1.x with AI nodes enabled.
- **Edge cases / potential failures:**
  - Claude may occasionally wrap JSON in markdown code fences or add preamble text — the downstream Parse Score node handles this with a regex fallback.
  - If the API key is missing or invalid, the node throws an authentication error.
  - Rate limits on the Anthropic API may cause 429 errors under high volume.
  - If Claude returns a score outside 1–10, the Route by Score node may not match any branch (see Block 1.3 edge cases).

---

**Node: Claude Sonnet**

- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1.3)
- **Technical role:** Language model sub-node providing Anthropic Claude access to the parent chain node.
- **Configuration:**
  - Model: `claude-sonnet-4-5`
  - Temperature: `0` (deterministic output for consistent scoring)
  - Credential: Anthropic API key (to be configured by user, obtained from `console.anthropic.com`)
- **Key expressions/variables:** None.
- **Input connections:** Connected as a sub-node to Score Lead Intent via the `ai_languageModel` input.
- **Output connections:** → Score Lead Intent (ai_languageModel, index 0)
- **Version-specific requirements:** lmChatAnthropic v1.3; requires n8n AI nodes package.
- **Edge cases / potential failures:**
  - Invalid or expired API key → 401 error.
  - Quota exceeded → 429 error.
  - Model name `claude-sonnet-4-5` must match an available model on the user's Anthropic account. If the model is renamed or deprecated, this must be updated.
  - To reduce cost at high volume, the model can be swapped to `claude-haiku` per the sticky note recommendation.

---

**Node: Parse Score**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical role:** Custom JavaScript code that parses the raw text output from the LLM chain, extracts the JSON object containing score/tier/reasoning, and merges these fields with the original lead data from the Extract Lead Fields node.
- **Configuration — JS Code:**
  1. Reads the raw text output from `$input.first().json.text`.
  2. Defaults to `{ score: 5, tier: 'Warm', reasoning: 'Could not parse AI response' }` as a safe fallback.
  3. Uses regex `/{[^}]+}/` to find the first JSON-like substring in the text.
  4. Parses it with `JSON.parse()`, wrapped in try/catch.
  5. Retrieves the original lead fields from the `Extract Lead Fields` node via `$('Extract Lead Fields').first().json`.
  6. Returns a merged object containing all lead fields plus `score`, `tier`, `reasoning`.
- **Key expressions/variables:** References `Extract Lead Fields` node by name for cross-node data access.
- **Input connections:** ← Score Lead Intent (main, index 0)
- **Output connections:** → Route by Score (main, index 0)
- **Edge cases / potential failures:**
  - If Claude returns malformed JSON (e.g., trailing commas, single quotes), `JSON.parse` fails and the fallback values (score 5, Warm) are used. This means the lead will be routed as Warm by default.
  - If the `Extract Lead Fields` node name is changed, the `$('Extract Lead Fields')` reference will break silently (returns undefined) causing a runtime error.
  - If Claude outputs multiple JSON objects, only the first match is captured.

---

#### Block 1.3 — Score-Based Routing

**Overview:** A Switch node evaluates the parsed numeric score and directs the execution into one of three output branches: Hot (score ≥ 7), Warm (score ≥ 4), or Cold (score ≥ 1). The rules are evaluated in order, so a score of 8 matches "Hot" first and does not fall through to "Warm" or "Cold."

**Nodes Involved:** Route by Score

---

**Node: Route by Score**

- **Type:** `n8n-nodes-base.switch` (v3.2)
- **Technical role:** Conditional router that splits execution into named output branches based on the `score` field.
- **Configuration — Rules:**

| Output Key | Condition | Left Value | Operator | Right Value |
|---|---|---|---|---|
| Hot | All conditions AND | `{{ $json.score }}` | Number ≥ | 7 |
| Warm | All conditions AND | `{{ $json.score }}` | Number ≥ | 4 |
| Cold | All conditions AND | `{{ $json.score }}` | Number ≥ | 1 |

- **Key expressions/variables:** `$json.score`
- **Input connections:** ← Parse Score (main, index 0)
- **Output connections:**
  - Output 0 (Hot) → Notify Team – Hot Lead (main, index 0)
  - Output 1 (Warm) → Warm Lead Reply (main, index 0)
  - Output 2 (Cold) → Cold Lead Reply (main, index 0)
- **Edge cases / potential failures:**
  - If `score` is missing or not a number (e.g., due to a Parse Score failure), none of the rules match and the item is silently dropped. This can be mitigated by adding a fallback/fallback output or ensuring the Parse Score fallback always returns a numeric score.
  - Score of 0 or negative would not match any rule (even Cold requires ≥ 1).
  - Score above 10 would match Hot (since ≥ 7), which is acceptable but indicates the AI returned an out-of-range value.

---

#### Block 1.4 — Hot Lead Branch (Score 7–10)

**Overview:** High-fit leads receive a three-step treatment: a Slack notification to the sales team, a personalised email reply encouraging a call with a calendar link, and a log entry in Google Sheets.

**Nodes Involved:** Notify Team – Hot Lead, Hot Lead Reply, Log to Sheets – Hot

---

**Node: Notify Team – Hot Lead**

- **Type:** `n8n-nodes-base.slack` (v2.2)
- **Technical role:** Sends a message to a Slack channel alerting the team about a hot lead.
- **Configuration:**
  - Channel: `#sales-leads` (by name)
  - Message template (key expressions):

    ```
    Hot lead just came in!

    Name: {{ $json.lead_name }}
    Company: {{ $json.lead_company }} ({{ $json.company_size }} employees)
    Email: {{ $json.lead_email }}
    Score: {{ $json.score }}/10
    Reason: {{ $json.reasoning }}

    Message: {{ $json.lead_message }}
    ```

  - Other options: default (none specified).
- **Key expressions/variables:** `$json.lead_name`, `$json.lead_company`, `$json.company_size`, `$json.lead_email`, `$json.score`, `$json.reasoning`, `$json.lead_message`
- **Input connections:** ← Route by Score, output 0 (Hot)
- **Output connections:** → Hot Lead Reply (main, index 0)
- **Edge cases / potential failures:**
  - Slack credential not connected → 401/403 error.
  - Channel `#sales-leads` does not exist or the bot lacks permission → error.
  - This node can be disabled (right-click → Disable) if Slack is not used. Disabling it will break the chain unless the connection is rewired from Route by Score directly to Hot Lead Reply.
  - Rate limiting if many hot leads arrive simultaneously.

---

**Node: Hot Lead Reply**

- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical role:** Sends a personalised email to the hot lead's email address.
- **Configuration:**
  - Email type: `text` (plain text, no HTML)
  - Send to: `{{ $json.lead_email }}`
  - Subject: `Re: Your enquiry — lets find a time to talk`
  - Message body (excerpt with expressions):

    ```
    Hi {{ $json.lead_name }},

    Thanks for reaching out — this is exactly the kind of challenge we love solving.

    Based on what you have shared, I think there is a real opportunity here. I would love to jump on a quick call to learn more about your situation and show you what is possible.

    You can book a time that works for you here: YOUR_CALENDAR_LINK

    Looking forward to connecting.

    Best,
    [Your Name]
    [Your Company]
    ```

  - Options: default (none specified).
- **Key expressions/variables:** `$json.lead_email`, `$json.lead_name`
- **Credential:** Gmail OAuth2 (must be configured by the user).
- **Input connections:** ← Notify Team – Hot Lead (main, index 0)
- **Output connections:** → Log to Sheets – Hot (main, index 0)
- **Edge cases / potential failures:**
  - `lead_email` is invalid or empty → Gmail API returns a 400 error.
  - Gmail daily send quota exceeded → 429/403 error.
  - `YOUR_CALENDAR_LINK` placeholder must be replaced by the user with an actual booking link.
  - `[Your Name]` and `[Your Company]` placeholders must be customised.

---

**Node: Log to Sheets – Hot**

- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Technical role:** Appends a row to a Google Sheet with all lead data plus the AI score and reasoning.
- **Configuration:**
  - Operation: `append`
  - Document ID: `YOUR_GOOGLE_SHEET_ID` (placeholder — must be replaced by user with the actual Google Sheets file ID)
  - Sheet name: `Sheet1`
  - Mapping mode: `defineBelow` (columns defined manually)
  - Column mapping:

| Column | Value |
|---|---|
| Name | `{{ $json.lead_name }}` |
| Email | `{{ $json.lead_email }}` |
| Company | `{{ $json.lead_company }}` |
| Size | `{{ $json.company_size }}` |
| Message | `{{ $json.lead_message }}` |
| Score | `{{ $json.score }}` |
| Tier | `Hot` (static string) |
| Reasoning | `{{ $json.reasoning }}` |
| Timestamp | `{{ $json.timestamp }}` |

- **Credential:** Google OAuth2 (must be configured by the user).
- **Input connections:** ← Hot Lead Reply (main, index 0)
- **Output connections:** None (terminal node in this branch).
- **Edge cases / potential failures:**
  - `YOUR_GOOGLE_SHEET_ID` not replaced → "Spreadsheet not found" error.
  - Google API rate limit or permission error if the sheet is not shared with the service account.
  - If column headers in the actual sheet don't match the mapped column names, the append will create new columns or fail depending on sheet settings.
  - If the sheet has additional required columns (e.g., formulas), the append may produce incomplete rows.

---

#### Block 1.5 — Warm Lead Branch (Score 4–6)

**Overview:** Promising leads that don't yet clearly fit receive a softer reply asking for more information, plus a log entry in Google Sheets for manual follow-up.

**Nodes Involved:** Warm Lead Reply, Log to Sheets – Warm

---

**Node: Warm Lead Reply**

- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical role:** Sends an exploratory email to the warm lead requesting more detail.
- **Configuration:**
  - Email type: `text`
  - Send to: `{{ $json.lead_email }}`
  - Subject: `Re: Your enquiry`
  - Message body (excerpt):

    ```
    Hi {{ $json.lead_name }},

    Thank you for getting in touch — great to hear from you.

    I have had a look at what you have shared and I would love to learn a bit more before we set up a call. Could you tell me a little more about the current challenge you are trying to solve and what a good outcome would look like for you?

    Looking forward to your reply.

    Best,
    [Your Name]
    [Your Company]
    ```

  - Options: default.
- **Key expressions/variables:** `$json.lead_email`, `$json.lead_name`
- **Credential:** Gmail OAuth2 (same credential as Hot Lead Reply; must be configured).
- **Input connections:** ← Route by Score, output 1 (Warm)
- **Output connections:** → Log to Sheets – Warm (main, index 0)
- **Edge cases / potential failures:** Same as Hot Lead Reply — invalid email, quota exceeded, placeholder text not customised.

---

**Node: Log to Sheets – Warm**

- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Technical role:** Appends a row to the same Google Sheet with Tier set to "Warm."
- **Configuration:** Identical to Log to Sheets – Hot, except:
  - Tier value: `Warm` (static string)
- **Credential:** Google OAuth2.
- **Input connections:** ← Warm Lead Reply (main, index 0)
- **Output connections:** None.
- **Edge cases / potential failures:** Same as Log to Sheets – Hot.

---

#### Block 1.6 — Cold Lead Branch (Score 1–3)

**Overview:** Low-fit leads receive a polite decline email so they still get a response, and their data is logged to Google Sheets for completeness.

**Nodes Involved:** Cold Lead Reply, Log to Sheets – Cold

---

**Node: Cold Lead Reply**

- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical role:** Sends a respectful decline email to the cold lead.
- **Configuration:**
  - Email type: `text`
  - Send to: `{{ $json.lead_email }}`
  - Subject: `Re: Your enquiry`
  - Message body (excerpt):

    ```
    Hi {{ $json.lead_name }},

    Thank you for reaching out and taking the time to get in touch.

    After reviewing your enquiry, I do not think we are the right fit for where you are right now — but I appreciate you considering us and wish you all the best in finding the right solution.

    If things change down the line, do not hesitate to reach back out.

    All the best,
    [Your Name]
    [Your Company]
    ```

  - Options: default.
- **Key expressions/variables:** `$json.lead_email`, `$json.lead_name`
- **Credential:** Gmail OAuth2.
- **Input connections:** ← Route by Score, output 2 (Cold)
- **Output connections:** → Log to Sheets – Cold (main, index 0)
- **Edge cases / potential failures:** Same email-related issues as other Gmail nodes.

---

**Node: Log to Sheets – Cold**

- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Technical role:** Appends a row to the same Google Sheet with Tier set to "Cold."
- **Configuration:** Identical to Log to Sheets – Hot, except:
  - Tier value: `Cold` (static string)
- **Credential:** Google OAuth2.
- **Input connections:** ← Cold Lead Reply (main, index 0)
- **Output connections:** None.
- **Edge cases / potential failures:** Same as Log to Sheets – Hot.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Inbound Lead Form | formTrigger v2.2 | Webhook entry point; n8n-native form collecting lead data | — (entry) | Extract Lead Fields | ## AI Lead Qualification Agent: Auto-Score, Route and Reply with Claude — Stop wasting time on leads that will never convert. This workflow reads every inbound form submission, uses Claude AI to score the lead 1-10 against your ideal customer profile, and automatically takes the right action — all without you lifting a finger. / ## Capture inbound lead — An n8n Form collects the lead's name, email, company, and message. No third-party form tool needed — the URL is generated automatically when you activate the workflow. |
| Extract Lead Fields | set v3.4 | Field normalisation and timestamp assignment | Inbound Lead Form | Score Lead Intent | ## Capture inbound lead — An n8n Form collects the lead's name, email, company, and message. No third-party form tool needed — the URL is generated automatically when you activate the workflow. |
| Score Lead Intent | chainLlm v1.4 | LangChain LLM chain that sends lead data + scoring prompt to Claude | Extract Lead Fields (main), Claude Sonnet (ai_languageModel) | Parse Score | ## Score lead with Claude AI — Claude Sonnet reads the submission and returns a score (1-10) plus a one-line reasoning. Edit the system prompt in this node to describe your ideal customer so Claude scores against your actual criteria. |
| Claude Sonnet | lmChatAnthropic v1.3 | Language model sub-node providing Anthropic Claude access | — (sub-node) | Score Lead Intent (ai_languageModel) | ## Score lead with Claude AI — Claude Sonnet reads the submission and returns a score (1-10) plus a one-line reasoning. Edit the system prompt in this node to describe your ideal customer so Claude scores against your actual criteria. |
| Parse Score | code v2 | Parses Claude's text output into structured score/tier/reasoning fields and merges with lead data | Score Lead Intent | Route by Score | ## Score lead with Claude AI — Claude Sonnet reads the submission and returns a score (1-10) plus a one-line reasoning. Edit the system prompt in this node to describe your ideal customer so Claude scores against your actual criteria. |
| Route by Score | switch v3.2 | Routes leads to Hot/Warm/Cold branches based on numeric score | Parse Score | Notify Team – Hot Lead (Hot), Warm Lead Reply (Warm), Cold Lead Reply (Cold) | ## Route by score — Splits leads into three tiers: Hot (7-10), Warm (4-6), and Cold (1-3). Each tier triggers a different set of actions below. |
| Notify Team – Hot Lead | slack v2.2 | Sends Slack channel alert with hot lead details | Route by Score (Hot) | Hot Lead Reply | ## Hot lead branch (score 7-10) — This lead matches your ideal profile. Sends an instant personalised reply email, fires a Slack alert to your team, and logs everything to Google Sheets. Slack is optional — right-click the Notify Team node and Disable if unused. |
| Hot Lead Reply | gmail v2.1 | Sends personalised email encouraging a call | Notify Team – Hot Lead | Log to Sheets – Hot | ## Hot lead branch (score 7-10) — This lead matches your ideal profile. Sends an instant personalised reply email, fires a Slack alert to your team, and logs everything to Google Sheets. Slack is optional — right-click the Notify Team node and Disable if unused. |
| Log to Sheets – Hot | googleSheets v4.5 | Appends hot lead data row to Google Sheets | Hot Lead Reply | — (terminal) | ## Hot lead branch (score 7-10) — This lead matches your ideal profile. Sends an instant personalised reply email, fires a Slack alert to your team, and logs everything to Google Sheets. Slack is optional — right-click the Notify Team node and Disable if unused. |
| Warm Lead Reply | gmail v2.1 | Sends exploratory email requesting more information | Route by Score (Warm) | Log to Sheets – Warm | ## Warm lead branch (score 4-6) — Promising but not yet a clear fit. Sends a friendly holding reply that sets expectations, then logs to Google Sheets for manual follow-up. |
| Log to Sheets – Warm | googleSheets v4.5 | Appends warm lead data row to Google Sheets | Warm Lead Reply | — (terminal) | ## Warm lead branch (score 4-6) — Promising but not yet a clear fit. Sends a friendly holding reply that sets expectations, then logs to Google Sheets for manual follow-up. |
| Cold Lead Reply | gmail v2.1 | Sends respectful decline email | Route by Score (Cold) | Log to Sheets – Cold | ## Cold lead branch (score 1-3) — Not a fit right now. Sends a short, respectful decline email so the lead gets a response and your inbox stays clean. Logged to Sheets for records. |
| Log to Sheets – Cold | googleSheets v4.5 | Appends cold lead data row to Google Sheets | Cold Lead Reply | — (terminal) | ## Cold lead branch (score 1-3) — Not a fit right now. Sends a short, respectful decline email so the lead gets a response and your inbox stays clean. Logged to Sheets for records. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "AI Lead Qualification Agent: Auto-Score, Route and Reply with Claude."

2. **Add the Form Trigger node:**
   - Node type: **Form Trigger** (`n8n-nodes-base.formTrigger`), version 2.2.
   - Form title: `Get in Touch`
   - Form description: `Tell us a bit about yourself and what you are looking for.`
   - Response mode: `lastNode`
   - Add five fields (all required):
     - `Full Name` — type: text
     - `Work Email` — type: email
     - `Company Name` — type: text
     - `Company Size` — type: dropdown, options: `1-10`, `11-50`, `51-200`, `201-500`, `500+`
     - `How can we help you?` — type: textarea
   - Name the node: **Inbound Lead Form**

3. **Add the Set node for field extraction:**
   - Node type: **Set** (`n8n-nodes-base.set`), version 3.4.
   - Name: **Extract Lead Fields**
   - Add six assignments:
     - `lead_name` (string) = `{{ $json['Full Name'] }}`
     - `lead_email` (string) = `{{ $json['Work Email'] }}`
     - `lead_company` (string) = `{{ $json['Company Name'] }}`
     - `company_size` (string) = `{{ $json['Company Size'] }}`
     - `lead_message` (string) = `{{ $json['How can we help you?'] }}`
     - `timestamp` (string) = `{{ new Date().toISOString() }}`
   - Connect: Inbound Lead Form → Extract Lead Fields

4. **Add the LangChain LLM Chain node:**
   - Node type: **Chain LLM** (`@n8n/n8n-nodes-langchain.chainLlm`), version 1.4.
   - Name: **Score Lead Intent**
   - Prompt type: `define`
   - Prompt text (paste exactly, then customise the Ideal Customer Profile section):

     ```
     You are a B2B lead qualification specialist. Score the following inbound lead from 1 to 10 based on how well they fit the ideal customer profile below.

     IDEAL CUSTOMER PROFILE:
     - Small to mid-size businesses (1-200 employees)
     - Looking for AI automation, workflow automation, or sales automation
     - Has a clear problem or goal mentioned
     - Decision maker or senior role

     LEAD DETAILS:
     Name: {{ $json.lead_name }}
     Company: {{ $json.lead_company }}
     Company Size: {{ $json.company_size }}
     Message: {{ $json.lead_message }}

     Respond ONLY with valid JSON in this exact format, nothing else:
     {"score": 8, "tier": "Hot", "reasoning": "One sentence explaining the score"}

     Score guide: 7-10 = Hot, 4-6 = Warm, 1-3 = Cold
     ```

   - Connect: Extract Lead Fields → Score Lead Intent (main input)

5. **Add the Anthropic Chat Model sub-node:**
   - Node type: **Chat Anthropic** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), version 1.3.
   - Name: **Claude Sonnet**
   - Model: `claude-sonnet-4-5`
   - Temperature: `0`
   - Credential: Create a new **Anthropic API** credential and paste your API key from `console.anthropic.com`.
   - Connect: Claude Sonnet → Score Lead Intent (ai_languageModel input). Drag the connector from Claude Sonnet's output into the small AI model connector on the Score Lead Intent node.

6. **Add the Code node to parse the AI response:**
   - Node type: **Code** (`n8n-nodes-base.code`), version 2.
   - Name: **Parse Score**
   - Language: JavaScript
   - Paste the following code:

     ```javascript
     const text = $input.first().json.text || '';
     let parsed = { score: 5, tier: 'Warm', reasoning: 'Could not parse AI response' };
     try {
       const match = text.match(/\{[^}]+\}/);
       if (match) parsed = JSON.parse(match[0]);
     } catch(e) {}
     const item = $('Extract Lead Fields').first().json;
     return {
       json: {
         lead_name:    item.lead_name,
         lead_email:   item.lead_email,
         lead_company: item.lead_company,
         company_size: item.company_size,
         lead_message: item.lead_message,
         timestamp:    item.timestamp,
         score:        parsed.score,
         tier:         parsed.tier,
         reasoning:    parsed.reasoning
       }
     };
     ```

   - Connect: Score Lead Intent → Parse Score

7. **Add the Switch node for routing:**
   - Node type: **Switch** (`n8n-nodes-base.switch`), version 3.2.
   - Name: **Route by Score**
   - Add three rules:
     - Rule 1 — Output key: `Hot`; Condition: `$json.score` (number) ≥ 7
     - Rule 2 — Output key: `Warm`; Condition: `$json.score` (number) ≥ 4
     - Rule 3 — Output key: `Cold`; Condition: `$json.score` (number) ≥ 1
   - Connect: Parse Score → Route by Score (main input)
   - Connect outputs:
     - Output 0 (Hot) → will connect to Notify Team – Hot Lead
     - Output 1 (Warm) → will connect to Warm Lead Reply
     - Output 2 (Cold) → will connect to Cold Lead Reply

8. **Add the Slack notification node (Hot branch):**
   - Node type: **Slack** (`n8n-nodes-base.slack`), version 2.2.
   - Name: **Notify Team – Hot Lead**
   - Credential: Create a new **Slack API** credential (OAuth2 or bot token) and grant access.
   - Channel selection: by name → `#sales-leads`
   - Message text:

     ```
     Hot lead just came in!

     Name: {{ $json.lead_name }}
     Company: {{ $json.lead_company }} ({{ $json.company_size }} employees)
     Email: {{ $json.lead_email }}
     Score: {{ $json.score }}/10
     Reason: {{ $json.reasoning }}

     Message: {{ $json.lead_message }}
     ```

   - Connect: Route by Score output 0 (Hot) → Notify Team – Hot Lead

9. **Add the Hot Lead Reply Gmail node:**
   - Node type: **Gmail** (`n8n-nodes-base.gmail`), version 2.1.
   - Name: **Hot Lead Reply**
   - Credential: Create a **Gmail OAuth2** credential and authenticate.
   - Email type: `text`
   - Send to: `{{ $json.lead_email }}`
   - Subject: `Re: Your enquiry — lets find a time to talk`
   - Message:

     ```
     Hi {{ $json.lead_name }},

     Thanks for reaching out — this is exactly the kind of challenge we love solving.

     Based on what you have shared, I think there is a real opportunity here. I would love to jump on a quick call to learn more about your situation and show you what is possible.

     You can book a time that works for you here: YOUR_CALENDAR_LINK

     Looking forward to connecting.

     Best,
     [Your Name]
     [Your Company]
     ```

   - **Important:** Replace `YOUR_CALENDAR_LINK`, `[Your Name]`, and `[Your Company]` with your actual values.
   - Connect: Notify Team – Hot Lead → Hot Lead Reply

10. **Add the Hot Lead Google Sheets logging node:**
    - Node type: **Google Sheets** (`n8n-nodes-base.googleSheets`), version 4.5.
    - Name: **Log to Sheets – Hot**
    - Credential: Create a **Google OAuth2** credential with Sheets access.
    - Operation: `append`
    - Document ID: Replace `YOUR_GOOGLE_SHEET_ID` with the ID of your actual Google Sheet (found in the sheet URL).
    - Sheet name: `Sheet1`
    - Mapping mode: `defineBelow`
    - Column mapping:
      - Name: `{{ $json.lead_name }}`
      - Email: `{{ $json.lead_email }}`
      - Company: `{{ $json.lead_company }}`
      - Size: `{{ $json.company_size }}`
      - Message: `{{ $json.lead_message }}`
      - Score: `{{ $json.score }}`
      - Tier: `Hot`
      - Reasoning: `{{ $json.reasoning }}`
      - Timestamp: `{{ $json.timestamp }}`
    - Connect: Hot Lead Reply → Log to Sheets – Hot

11. **Add the Warm Lead Reply Gmail node:**
    - Node type: **Gmail** (`n8n-nodes-base.gmail`), version 2.1.
    - Name: **Warm Lead Reply**
    - Credential: Select the same Gmail credential.
    - Email type: `text`
    - Send to: `{{ $json.lead_email }}`
    - Subject: `Re: Your enquiry`
    - Message:

      ```
      Hi {{ $json.lead_name }},

      Thank you for getting in touch — great to hear from you.

      I have had a look at what you have shared and I would love to learn a bit more before we set up a call. Could you tell me a little more about the current challenge you are trying to solve and what a good outcome would look like for you?

      Looking forward to your reply.

      Best,
      [Your Name]
      [Your Company]
      ```

    - **Important:** Replace `[Your Name]` and `[Your Company]`.
    - Connect: Route by Score output 1 (Warm) → Warm Lead Reply

12. **Add the Warm Lead Google Sheets logging node:**
    - Node type: **Google Sheets** (`n8n-nodes-base.googleSheets`), version 4.5.
    - Name: **Log to Sheets – Warm**
    - Same configuration as Log to Sheets – Hot, except:
      - Tier: `Warm`
    - Connect: Warm Lead Reply → Log to Sheets – Warm

13. **Add the Cold Lead Reply Gmail node:**
    - Node type: **Gmail** (`n8n-nodes-base.gmail`), version 2.1.
    - Name: **Cold Lead Reply**
    - Credential: Select the same Gmail credential.
    - Email type: `text`
    - Send to: `{{ $json.lead_email }}`
    - Subject: `Re: Your enquiry`
    - Message:

      ```
      Hi {{ $json.lead_name }},

      Thank you for reaching out and taking the time to get in touch.

      After reviewing your enquiry, I do not think we are the right fit for where you are right now — but I appreciate you considering us and wish you all the best in finding the right solution.

      If things change down the line, do not hesitate to reach back out.

      All the best,
      [Your Name]
      [Your Company]
      ```

    - **Important:** Replace `[Your Name]` and `[Your Company]`.
    - Connect: Route by Score output 2 (Cold) → Cold Lead Reply

14. **Add the Cold Lead Google Sheets logging node:**
    - Node type: **Google Sheets** (`n8n-nodes-base.googleSheets`), version 4.5.
    - Name: **Log to Sheets – Cold**
    - Same configuration as Log to Sheets – Hot, except:
      - Tier: `Cold`
    - Connect: Cold Lead Reply → Log to Sheets – Cold

15. **Prepare the Google Sheet:**
    - Create a new Google Sheet (or use an existing one).
    - Name the first sheet tab `Sheet1`.
    - Add the following headers in row 1: `Timestamp`, `Name`, `Email`, `Company`, `Size`, `Message`, `Score`, `Tier`, `Reasoning`.
    - Copy the sheet's ID from the URL: `https://docs.google.com/spreadsheets/d/YOUR_GOOGLE_SHEET_ID/edit`
    - Paste this ID into all three Google Sheets nodes (Log to Sheets – Hot/Warm/Cold) replacing `YOUR_GOOGLE_SHEET_ID`.

16. **Verify all credentials:**
    - **Anthropic API**: Key pasted into Claude Sonnet credential; tested at `console.anthropic.com`.
    - **Gmail OAuth2**: Authenticated in all three Gmail nodes (Hot Lead Reply, Warm Lead Reply, Cold Lead Reply).
    - **Google OAuth2**: Authenticated in all three Google Sheets nodes with Drive/Sheets scope.
    - **Slack**: Authenticated in Notify Team – Hot Lead; channel `#sales-leads` exists and the bot has access. If Slack is not needed, right-click the node and select **Disable**, then rewire: Route by Score output 0 → Hot Lead Reply directly.

17. **Customise the scoring prompt:**
    - Open the **Score Lead Intent** node and edit the `IDEAL CUSTOMER PROFILE` section to match your actual target customer criteria. Adjust the score guide thresholds in the prompt text if you change the routing thresholds.

18. **Adjust routing thresholds (optional):**
    - Open **Route by Score** and change the numeric boundaries if your business defines Hot/Warm/Cold differently. Ensure the prompt's score guide matches the thresholds you set here.

19. **Optionally swap to a cheaper model:**
    - To reduce API costs at high volume, open **Claude Sonnet** and change the model from `claude-sonnet-4-5` to `claude-haiku`.

20. **Activate the workflow and test:**
    - Click **Activate** to enable the form webhook.
    - Copy the form URL from the **Inbound Lead Form** node.
    - Submit three test entries (one expected Hot, one Warm, one Cold) to verify all branches.
    - Check Gmail sent folders, Slack channel, and Google Sheet rows for correctness.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The form URL is auto-generated when the workflow is activated. Deactivating the workflow disables the form. | Inbound Lead Form setup |
| Claude Sonnet is called with temperature 0 for deterministic, repeatable scoring. Increase temperature only if you intentionally want scoring variation. | Claude Sonnet configuration |
| The Parse Score node uses a fallback of score 5 / Warm if Claude's response cannot be parsed. This ensures no lead is dropped silently but means parsing failures route to the Warm branch. | Parse Score edge case |
| If Slack is not used, disable the Notify Team – Hot Lead node and reconnect Route by Score output 0 directly to Hot Lead Reply. Otherwise the Hot branch will stop at the disabled node. | Slack optional — workflow design |
| Google Sheets column headers in the physical sheet must exactly match the mapped column names (Timestamp, Name, Email, Company, Size, Message, Score, Tier, Reasoning) for clean appends. | Google Sheets setup |
| Replace all placeholder strings before activating: `YOUR_CALENDAR_LINK`, `YOUR_GOOGLE_SHEET_ID`, `[Your Name]`, `[Your Company]` | Throughout the workflow |
| For high-volume usage, swap `claude-sonnet-4-5` to `claude-haiku` in the Claude Sonnet sub-node to reduce API costs | Customisation — sticky note |
| Adjust hot/warm/cold score thresholds in the Route by Score node to match your own standards, and keep the scoring prompt's score guide aligned with those thresholds | Customisation — sticky note |
| Anthropic API key obtained from console.anthropic.com | Credential setup |
| The three Gmail nodes can share a single Gmail OAuth2 credential; there is no need to create separate credentials per node | Credential efficiency |
| The three Google Sheets nodes also share a single Google OAuth2 credential | Credential efficiency |
| This workflow uses execution order v1 (`"executionOrder":"v1"`). If migrating to an n8n instance using v0 execution order, re-test all branches to confirm data flows as expected. | n8n settings |