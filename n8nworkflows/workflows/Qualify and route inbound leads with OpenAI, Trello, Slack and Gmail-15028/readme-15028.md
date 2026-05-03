Qualify and route inbound leads with OpenAI, Trello, Slack and Gmail

https://n8nworkflows.xyz/workflows/qualify-and-route-inbound-leads-with-openai--trello--slack-and-gmail-15028


# Qualify and route inbound leads with OpenAI, Trello, Slack and Gmail

Let me now write the comprehensive document.### 1. Workflow Overview

**"Speed to Lead"** is an automated inbound lead qualification and routing system. It captures lead information from a web form, uses an OpenAI-powered AI Agent to classify each lead as **HOT**, **WARM**, or **COLD** based on budget and timeline urgency, then routes the lead to the appropriate follow-up channel:

- **HOT leads** → Slack channel alert (immediate call-to-action for the sales team)
- **WARM leads** → Gmail message acknowledging the inquiry (24-hour follow-up promise)
- **COLD leads** → Gmail nurture email to the lead (soft engagement, invite to book a call)

**Logical Blocks:**

| Block | Purpose |
|---|---|
| **1.1 Lead Capture** | Web form receives and validates lead data (name, email, phone, budget, timeline, service). |
| **1.2 AI Qualification** | LangChain Agent with OpenAI model + Structured Output Parser classifies the lead into HOT / WARM / COLD. |
| **1.3 Data Normalization** | Edit Fields flattens AI output and merges it with original form data into a single, clean item. |
| **1.4 Lead Routing** | Switch node dispatches the lead to the correct output branch based on the AI score. |
| **1.5 Action Execution** | Slack notification (HOT), Gmail acknowledgment (WARM), or Gmail nurture email (COLD). |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Lead Capture

**Overview:** A public n8n form trigger collects structured lead information and initiates the workflow on every submission.

**Nodes Involved:**
- On form submission

**Node Details:**

| Attribute | Detail |
|---|---|
| **Node Name** | On form submission |
| **Type** | `n8n-nodes-base.formTrigger` (v2.5) |
| **Technical Role** | Entry-point trigger; exposes a hosted form and emits one item per submission |
| **Configuration** | Form Title: *"What do you want us to help?"*; Description: *"We'll be in touch ASAP"* |
| **Form Fields** | 1) Full Name (text) 2) Email (email) 3) Phone Number (number) 4) Budget (dropdown: `$1,000 – $5,000` / `$5,000 – $10,000`) 5) Timeline (dropdown: `ASAP / This week` / `Within a month` / `In 2–3 months` / `Just exploring`) 6) Service Needed (text) |
| **Key Expressions** | None — all values are literal form field definitions |
| **Input Connections** | None (entry point) |
| **Output Connections** | → AI Agent (main, index 0) |
| **Version Requirements** | formTrigger v2.5 or later |
| **Edge Cases / Failures** | Empty or malformed submissions (e.g., invalid email format) are not explicitly validated; Budget dropdown is limited to two options, so leads with budgets outside those ranges cannot self-select. Phone field type is `number` which may reject entries with formatting (dashes, plus signs). |

---

#### Block 1.2 — AI Qualification

**Overview:** A LangChain Agent receives the form data as a system message, instructs OpenAI's GPT-4.1-mini to classify the lead, and uses a Structured Output Parser to guarantee a machine-readable response.

**Nodes Involved:**
- AI Agent
- OpenAI Chat Model
- Structured Output Parser

**Node Details:**

---

**AI Agent**

| Attribute | Detail |
|---|---|
| **Node Name** | AI Agent |
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v3.1) |
| **Technical Role** | Orchestrates the LLM call; combines the prompt, language model, and output parser |
| **Prompt Type** | `define` (user-defined system message) |
| **System Message Expression** | Combines all form fields into a structured prompt:<br>`Name: {{ $json["Full Name"] }}`<br>`Email: {{ $json["Email"] }}`<br>`Budget: {{ $json["Budget"] }}`<br>`Timeline: {{ $json["Timeline"] }}`<br>`Service: {{ $json["Service Needed"] }}` |
| **User Prompt (Text)** | Hard-coded classification rules: "You are a lead qualification assistant…" Returns ONLY one word: HOT, WARM, or COLD, based on budget ≥ $5,000 + ASAP for HOT, budget $1,000–$5,000 OR timeline within a month for WARM, budget under $1,000 OR just exploring for COLD. |
| **hasOutputParser** | `true` — the Structured Output Parser is connected |
| **Input Connections** | ← On form submission (main, index 0); ← OpenAI Chat Model (ai_languageModel, index 0); ← Structured Output Parser (ai_outputParser, index 0) |
| **Output Connections** | → Edit Fields (main, index 0) |
| **Output Structure** | Returns an `output` object containing `score` (HOT/WARM/COLD) and `reason` (short explanation) |
| **Edge Cases / Failures** | If the LLM returns extra text beyond the single word, the parser may still extract the structured fields, but malformed JSON could cause a parsing error. Rate limits or auth failures on the OpenAI API will halt execution. The "ONLY one word" instruction conflicts with the Structured Output Parser which expects a JSON object — the parser should win, but inconsistent prompting could cause unexpected model behavior. |

---

**OpenAI Chat Model**

| Attribute | Detail |
|---|---|
| **Node Name** | OpenAI Chat Model |
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) |
| **Technical Role** | Provides the LLM backend (GPT-4.1-mini) to the AI Agent |
| **Model** | `gpt-4.1-mini` (selected from list) |
| **Built-in Tools** | None enabled |
| **Options** | Default (no temperature, max tokens, or other overrides set) |
| **Input Connections** | None (sub-node, connected to AI Agent) |
| **Output Connections** | → AI Agent (ai_languageModel, index 0) |
| **Credential Requirement** | OpenAI API key credential configured in n8n |
| **Edge Cases / Failures** | Invalid or expired API key; quota exceeded; model name not available in the user's OpenAI organization; network timeouts. |

---

**Structured Output Parser**

| Attribute | Detail |
|---|---|
| **Node Name** | Structured Output Parser |
| **Type** | `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) |
| **Technical Role** | Forces the LLM to return a JSON object matching the provided schema |
| **JSON Schema Example** | `{ "score": "HOT | WARM | COLD", "reason": "short explanation" }` |
| **Input Connections** | None (sub-node, connected to AI Agent) |
| **Output Connections** | → AI Agent (ai_outputParser, index 0) |
| **Edge Cases / Failures** | If the model fails to produce valid JSON matching the schema, the parser will throw a parsing error. The schema example does not enforce `enum` constraints — any string could be returned for `score`, potentially leading to a Switch mismatch downstream. |

---

#### Block 1.3 — Data Normalization

**Overview:** Flattens the nested AI output and merges it with the original form submission data, producing a clean item with all fields at the top level for downstream consumption.

**Nodes Involved:**
- Edit Fields

**Node Details:**

| Attribute | Detail |
|---|---|
| **Node Name** | Edit Fields |
| **Type** | `n8n-nodes-base.set` (v3.4) |
| **Technical Role** | Field mapping / reshaping node |
| **Assignments** | 1) `Score` (string) = `{{ $json.output.score }}` 2) `reason` (string) = `{{ $json.output.reason }}` 3) `Name` (string) = `{{ $('On form submission').item.json['Full Name'] }}` 4) `Email` (string) = `{{ $('On form submission').item.json.Email }}` 5) `Phone` (number) = `{{ $('On form submission').item.json['Phone Number'] }}` 6) `budget` (string) = `{{ $('On form submission').item.json.Budget }}` 7) `Service needed` (string) = `{{ $('On form submission').item.json['Service Needed'] }}` |
| **includeOtherFields** | `true` — original AI Agent output (including `output.score`, `output.reason`) is retained alongside new assignments |
| **Input Connections** | ← AI Agent (main, index 0) |
| **Output Connections** | → Switch (main, index 0) |
| **Key Behavior** | Because `includeOtherFields` is true, `$json.output.score` remains accessible in downstream nodes (used by the Switch). The new top-level `Score` field is also available. |
| **Edge Cases / Failures** | If the AI Agent output structure changes (e.g., no `output.score`), the expressions will resolve to `undefined`. The Phone field is typed as `number`, which strips leading zeros or formatting — problematic for international phone numbers. |

---

#### Block 1.4 — Lead Routing

**Overview:** Routes the lead to one of three action branches by matching the AI score against literal values.

**Nodes Involved:**
- Switch

**Node Details:**

| Attribute | Detail |
|---|---|
| **Node Name** | Switch |
| **Type** | `n8n-nodes-base.switch` (v3.4) |
| **Technical Role** | Conditional router; 3 output branches |
| **Rule 0 (Output 0 — HOT)** | `{{$json.output.score}}` equals `"HOT"` (string, case-sensitive, strict type validation) |
| **Rule 1 (Output 1 — WARM)** | `{{$json.output.score}}` equals `"WARM"` |
| **Rule 2 (Output 2 — COLD)** | `{{$json.output.score}}` equals `"COLD"` |
| **Options** | Default (no fallback configured; unmatched items are dropped) |
| **Input Connections** | ← Edit Fields (main, index 0) |
| **Output Connections** | Output 0 → Send a message (Slack); Output 1 → Send a message1 (Gmail); Output 2 → Send a message2 (Gmail) |
| **Edge Cases / Failures** | If the AI returns a score other than HOT/WARM/COLD (e.g., lowercase "hot" or extra whitespace), no rule matches and the lead is silently dropped — no error is raised. There is no fallback or default branch. The expression references `$json.output.score` (nested), not `$json.Score` (top-level), which works only because `includeOtherFields` is true. |

---

#### Block 1.5 — Action Execution

**Overview:** Three parallel terminal branches execute the appropriate response: a Slack alert for HOT leads, a professional acknowledgment email for WARM leads, and a softer nurture email for COLD leads.

**Nodes Involved:**
- Send a message (Slack — HOT)
- Send a message1 (Gmail — WARM)
- Send a message2 (Gmail — COLD)

**Node Details:**

---

**Send a message (Slack — HOT)**

| Attribute | Detail |
|---|---|
| **Node Name** | Send a message |
| **Type** | `n8n-nodes-base.slack` (v2.4) |
| **Technical Role** | Posts an urgent lead alert to a Slack channel |
| **Channel** | User-selected channel (placeholder `<SLACK_CHANNEL_ID>` must be replaced) |
| **Channel Selection Mode** | `list` (picker) |
| **Authentication** | OAuth2 |
| **Message Template** | `🔥 HOT LEAD — Call NOW` followed by Name, Email, Phone, Budget, Timeline (mapped from `$json.reason` — see note), Service |
| **Key Expressions** | `$json.Name`, `$json.Email`, `$json.Phone`, `$json.budget`, `$json.reason`, `$json['Service needed']` |
| **Input Connections** | ← Switch output 0 |
| **Output Connections** | None (terminal) |
| **Edge Cases / Failures** | The Timeline field in the Slack message is populated from `$json.reason` (the AI's short explanation), not the original form Timeline value. The original Timeline is not mapped in Edit Fields. If the Slack OAuth2 token is revoked or the channel ID is invalid, the node will fail. Channel ID placeholder must be manually replaced. |

---

**Send a message1 (Gmail — WARM)**

| Attribute | Detail |
|---|---|
| **Node Name** | Send a message1 |
| **Type** | `n8n-nodes-base.gmail` (v2.2) |
| **Technical Role** | Sends a professional acknowledgment email for WARM leads |
| **Send To** | `<YOUR_TEAM_EMAIL>` — a placeholder that must be replaced with a real email address |
| **Subject** | `We received your inquiry` |
| **Email Type** | Plain text |
| **Message Template** | Addressed to the lead by first name; thanks them for their inquiry about the selected service; promises follow-up within 24 hours; signed with `[Your Name / Company]` placeholder |
| **Key Expressions** | `$('On form submission').item.json['Full Name']`, `$('On form submission').item.json['Service Needed']` |
| **Input Connections** | ← Switch output 1 |
| **Output Connections** | None (terminal) |
| **Credential Requirement** | Gmail OAuth2 credential configured in n8n |
| **Edge Cases / Failures** | The `sendTo` address is a team/internal email, but the message body is written as if addressed to the lead. This may be intentional (internal notification with template) or a configuration oversight where the lead's email (`$json.Email`) should be used instead. The `[Your Name / Company]` signature placeholder must be manually replaced. |

---

**Send a message2 (Gmail — COLD)**

| Attribute | Detail |
|---|---|
| **Node Name** | Send a message2 |
| **Type** | `n8n-nodes-base.gmail` (v2.2) |
| **Technical Role** | Sends a soft nurture email to COLD leads |
| **Send To** | `={{ $json.Email }}` — dynamically uses the lead's email from Edit Fields |
| **Subject** | `Thanks for your interest!` |
| **Email Type** | Plain text |
| **Message Template** | Casual tone; thanks for reaching out; invites them to reply or book a free call; signed `Raejan | CoreFlow` |
| **Key Expressions** | `$('On form submission').item.json['Full Name']` for salutation |
| **Input Connections** | ← Switch output 2 |
| **Output Connections** | None (terminal) |
| **Credential Requirement** | Gmail OAuth2 credential (same as Send a message1, or separate) |
| **Edge Cases / Failures** | If `$json.Email` is empty or invalid, the Gmail node will throw a delivery error. The signature is hard-coded as `Raejan | CoreFlow` — should be updated if re-branding. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | formTrigger (v2.5) | Entry-point trigger; collects lead data via hosted form | — | AI Agent | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| AI Agent | @n8n/n8n-nodes-langchain.agent (v3.1) | LLM orchestrator; classifies lead as HOT/WARM/COLD | On form submission (main), OpenAI Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser) | Edit Fields | Use the PROMPT: You are a lead qualification assistant. Analyze the lead data and return ONLY one word: HOT, WARM, or COLD. Rules: - HOT: Budget is $5,000 or above AND timeline is ASAP or this week. High motivation. - WARM: Budget is $1,000–$5,000 OR timeline is within a month. Interested but not urgent. - COLD: Budget under $1,000 OR just exploring OR vague answers. Return ONLY the single word. No explanation. No punctuation. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | LLM backend providing GPT-4.1-mini | — (sub-node) | AI Agent (ai_languageModel) | Use the PROMPT: You are a lead qualification assistant. Analyze the lead data and return ONLY one word: HOT, WARM, or COLD. Rules: - HOT: Budget is $5,000 or above AND timeline is ASAP or this week. High motivation. - WARM: Budget is $1,000–$5,000 OR timeline is within a month. Interested but not urgent. - COLD: Budget under $1,000 OR just exploring OR vague answers. Return ONLY the single word. No explanation. No punctuation. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured (v1.3) | Enforces JSON output schema for AI response | — (sub-node) | AI Agent (ai_outputParser) | Use the PROMPT: You are a lead qualification assistant. Analyze the lead data and return ONLY one word: HOT, WARM, or COLD. Rules: - HOT: Budget is $5,000 or above AND timeline is ASAP or this week. High motivation. - WARM: Budget is $1,000–$5,000 OR timeline is within a month. Interested but not urgent. - COLD: Budget under $1,000 OR just exploring OR vague answers. Return ONLY the single word. No explanation. No punctuation. |
| Edit Fields | set (v3.4) | Flattens AI output and merges with form data | AI Agent | Switch | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| Switch | switch (v3.4) | Routes lead to HOT/WARM/COLD action branch | Edit Fields | Send a message (output 0), Send a message1 (output 1), Send a message2 (output 2) | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| Send a message | slack (v2.4) | Posts HOT lead alert to Slack channel | Switch (output 0) | — | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| Send a message1 | gmail (v2.2) | Sends WARM lead acknowledgment email | Switch (output 1) | — | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| Send a message2 | gmail (v2.2) | Sends COLD lead nurture email | Switch (output 2) | — | Try it out! This n8n workflow automates a speed-to-lead system, capturing form submissions and quickly routing qualified leads to tools like Trello and Gmail for fast follow-up. How it works: Triggers on form submission, feeding data into an AI Agent powered by an OpenAI Chat Model with chat memory. The AI processes leads via a structured output parser, then a switch (rules mode) decides actions like editing fields or sending messages. Outputs route to Trello (task creation), Gmail (notifications/emails), and possibly other apps for lead nurturing. How to use: Import the workflow into your n8n instance and connect credentials for OpenAI, Trello, Gmail, and your form tool (e.g., Google Forms). Test by submitting a form; monitor the AI agent for lead qualification and verify outputs in Trello/Gmail. Activate the workflow to run live, ensuring webhooks or triggers are set for production forms. Requirements: n8n account (self-hosted or cloud); API keys for OpenAI, Trello, Gmail OAuth2. Form tool integration (e.g., Google Forms webhook). For verified creator status: Submit 2-3 approved public workflows via n8n Creator Portal, including this one, to gain the badge and directory listing |
| Sticky Note | stickyNote (v1) | Informational: usage instructions and requirements | — | — | (content embedded in rows above) |
| Sticky Note1 | stickyNote (v1) | Informational: AI prompt reference | — | — | (content embedded in rows above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and name it "Speed to Lead".

2. **Add the Form Trigger node:**
   - Type: `Form Trigger` (`n8n-nodes-base.formTrigger`)
   - Form Title: `What do you want us to help?`
   - Form Description: `We'll be in touch ASAP`
   - Add fields in this order:
     - **Full Name** — type: Text
     - **Email** — type: Email
     - **Phone Number** — type: Number
     - **Budget** — type: Dropdown, options: `$1,000 – $5,000` and `$5,000 – $10,000`
     - **Timeline** — type: Dropdown, options: `ASAP / This week`, `Within a month`, `In 2–3 months`, `Just exploring`
     - **Service Needed** — type: Text
   - Leave Options at defaults.

3. **Add the AI Agent node:**
   - Type: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`)
   - Prompt Type: `Define`
   - System Message (expression):
     ```
     Name: {{ $json["Full Name"] }}
     Email: {{ $json["Email"] }}
     Budget: {{ $json["Budget"] }}
     Timeline: {{ $json["Timeline"] }}
     Service: {{ $json["Service Needed"] }}
     ```
   - Text (user prompt):
     ```
     You are a lead qualification assistant. Analyze the lead data and return ONLY one word: HOT, WARM, or COLD.

     Rules:
     - HOT: Budget is $5,000 or above AND timeline is ASAP or this week. High motivation.
     - WARM: Budget is $1,000–$5,000 OR timeline is within a month. Interested but not urgent.
     - COLD: Budget under $1,000 OR just exploring OR vague answers.

     Return ONLY the single word. No explanation. No punctuation.
     ```
   - Set `hasOutputParser` to `true`.

4. **Add the OpenAI Chat Model sub-node:**
   - Type: `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
   - Model: select `gpt-4.1-mini` from the list
   - Leave Options and Built-in Tools at defaults
   - Configure an **OpenAI API key** credential in n8n and assign it to this node
   - Connect its `ai_languageModel` output to the AI Agent's `ai_languageModel` input.

5. **Add the Structured Output Parser sub-node:**
   - Type: `Structured Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`)
   - JSON Schema Example:
     ```json
     {
       "score": "HOT | WARM | COLD",
       "reason": "short explanation"
     }
     ```
   - Connect its `ai_outputParser` output to the AI Agent's `ai_outputParser` input.

6. **Connect Form Trigger → AI Agent** (main output index 0 → main input index 0).

7. **Add the Edit Fields (Set) node:**
   - Type: `Edit Fields (Set)` (`n8n-nodes-base.set`)
   - Set `includeOtherFields` to `true`
   - Add assignments:
     | Field Name | Type | Value Expression |
     |---|---|---|
     | Score | String | `={{ $json.output.score }}` |
     | reason | String | `={{ $json.output.reason }}` |
     | Name | String | `={{ $('On form submission').item.json['Full Name'] }}` |
     | Email | String | `={{ $('On form submission').item.json.Email }}` |
     | Phone | Number | `={{ $('On form submission').item.json['Phone Number'] }}` |
     | budget | String | `={{ $('On form submission').item.json.Budget }}` |
     | Service needed | String | `={{ $('On form submission').item.json['Service Needed'] }}` |
   - Connect AI Agent main output → Edit Fields main input.

8. **Add the Switch node:**
   - Type: `Switch` (`n8n-nodes-base.switch`)
   - Add three rules:
     - **Rule 0:** Condition: `{{ $json.output.score }}` equals `HOT` (string, case-sensitive, strict type validation)
     - **Rule 1:** Condition: `{{ $json.output.score }}` equals `WARM`
     - **Rule 2:** Condition: `{{ $json.output.score }}` equals `COLD`
   - Leave fallback disabled.
   - Connect Edit Fields main output → Switch main input.

9. **Add the Slack node (HOT leads):**
   - Type: `Slack` (`n8n-nodes-base.slack`)
   - Operation: Send a message
   - Channel: Select your target channel from the list (replace the placeholder)
   - Authentication: OAuth2 (configure a Slack OAuth2 credential in n8n)
   - Message (expression):
     ```
     🔥 HOT LEAD — Call NOW

     Name: {{ $json.Name }}
     Email: {{ $json.Email }}
     Phone: {{ $json.Phone }}
     Budget: {{ $json.budget }}
     Timeline: {{ $json.reason }}
     Service: {{ $json['Service needed'] }}
     ```
   - Connect Switch output 0 → this node's main input.

10. **Add the Gmail node for WARM leads:**
    - Type: `Gmail` (`n8n-nodes-base.gmail`)
    - Operation: Send
    - Send To: Replace `<YOUR_TEAM_EMAIL>` with your actual team/lead email address (or use `={{ $json.Email }}` to send directly to the lead)
    - Subject: `We received your inquiry`
    - Email Type: Text
    - Message (expression):
      ```
      Hi {{ $('On form submission').item.json['Full Name'] }},

      Thank you for reaching out! We've received your inquiry about {{ $('On form submission').item.json['Service Needed'] }}.

      One of our team members will be in touch with you within 24 hours.

      Best regards,
      [Your Name / Company]
      ```
    - Configure a **Gmail OAuth2** credential and assign it.
    - Connect Switch output 1 → this node's main input.

11. **Add the Gmail node for COLD leads:**
    - Type: `Gmail` (`n8n-nodes-base.gmail`)
    - Operation: Send
    - Send To (expression): `={{ $json.Email }}`
    - Subject: `Thanks for your interest!`
    - Email Type: Text
    - Message (expression):
      ```
      Hi {{ $('On form submission').item.json['Full Name'] }},  

      Thanks for reaching out! We'd love to learn more about what you're looking for.  

      When you're ready to take the next step, feel free to reply to this email or book a free call with us.  

      Best regards, 
      Raejan | CoreFlow
      ```
    - Assign the same or a separate Gmail OAuth2 credential.
    - Connect Switch output 2 → this node's main input.

12. **Replace all placeholders:**
    - `<SLACK_CHANNEL_ID>` → your actual Slack channel ID
    - `<YOUR_TEAM_EMAIL>` → your team's email address (or the lead's email)
    - `[Your Name / Company]` → your real signature
    - `Raejan | CoreFlow` → update if different branding is needed

13. **Test the workflow:**
    - Open the form URL generated by the Form Trigger node
    - Submit test entries for each scenario:
      - Budget `$5,000 – $10,000` + Timeline `ASAP / This week` → expect HOT
      - Budget `$1,000 – $5,000` + Timeline `Within a month` → expect WARM
      - Budget `$1,000 – $5,000` + Timeline `Just exploring` → expect COLD
    - Verify Slack message, Gmail messages, and AI classification in each run

14. **Activate the workflow** using the toggle in n8n to make it live.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The main sticky note references **Trello** as an output destination, but **no Trello node exists** in this workflow. If Trello integration is desired (e.g., creating a card for HOT leads), a Trello node must be added manually. | Workflow sticky note |
| The **Phone Number** form field is typed as `number`, which cannot capture international formats with `+`, spaces, or dashes. Consider changing the field type to `text` for real-world use. | Form field configuration |
| The Slack message maps **Timeline** from `$json.reason` (the AI's explanation) instead of the original form Timeline value. The original Timeline is not passed through Edit Fields. This may be intentional but should be verified. | Slack node template |
| The **WARM lead Gmail** node sends to `<YOUR_TEAM_EMAIL>` with a message addressed to the lead. Decide whether this is an internal notification or a direct lead email and update `sendTo` accordingly. | Send a message1 node |
| The Structured Output Parser schema uses a free-text `"score"` field rather than a strict enum. If the model hallucinates a value outside HOT/WARM/COLD, the Switch will silently drop the lead. Consider adding a fallback branch or a schema enum constraint. | Switch / Parser design |
| The AI Agent prompt says "Return ONLY the single word" but the Structured Output Parser expects a JSON object with `score` and `reason`. These instructions conflict; the parser enforces JSON, so the "single word" instruction is effectively overridden. Clean up the prompt to explicitly request the JSON format. | AI Agent prompt |
| Gmail nodes require **OAuth2** credentials. Ensure the Google Cloud project has the Gmail API enabled and the OAuth consent screen configured. | Gmail credential setup |
| Slack node requires **OAuth2** with appropriate scopes (`chat:write`, `channels:read`). Install the Slack app in your workspace and authorize it in n8n. | Slack credential setup |
| For n8n verified creator status: submit 2–3 approved public workflows via the n8n Creator Portal to gain the badge and directory listing. | Sticky note reference |
| Branding in COLD lead email is hard-coded as **Raejan | CoreFlow**. Update for your own organization. | Send a message2 signature |