Route and triage support tickets with Claude Sonnet and your CRM

https://n8nworkflows.xyz/workflows/route-and-triage-support-tickets-with-claude-sonnet-and-your-crm-14826


# Route and triage support tickets with Claude Sonnet and your CRM

# 1. Workflow Overview

This workflow automates support ticket intake, AI-based triage, routing, CRM/helpdesk updating, escalation, and post-processing observability logging.

It is designed for teams that receive support requests from multiple channels and want to:
- ingest tickets from email or HTTP webhook
- normalize inconsistent payloads
- translate non-English messages into English
- classify ticket urgency, sentiment, category, and churn risk with Claude Sonnet
- route tickets either to an auto-reply/CRM update path or an escalation path
- log operational metrics for monitoring

## 1.1 Input Reception

The workflow starts from two entry points:
- **Email Trigger (IMAP)** for inbound email tickets
- **Webhook Trigger** for programmatic ticket submission via HTTP POST

Both sources converge into a shared processing path.

## 1.2 Configuration and Payload Preparation

A configuration node injects endpoint values used later in the workflow. The incoming content is then cleaned and normalized so downstream AI and routing nodes receive a consistent ticket shape.

## 1.3 Language Detection and Translation

An AI agent detects the ticket language and translates the message into English if necessary, returning structured JSON with language metadata and confidence.

## 1.4 Support Intelligence Classification

A second AI agent analyzes the English text and produces structured triage fields such as sentiment, urgency, category, summary, churn risk, and recommended handling path.

## 1.5 Decision Routing

A Switch node routes tickets based on AI output:
- **Auto Reply** when `recommended_path = auto_reply`
- **Escalate** when urgency is critical or `recommended_path = escalate`
- fallback also goes to escalation

## 1.6 Auto Reply and CRM Update

For tickets suitable for direct handling, the workflow generates a draft response using Claude Sonnet and posts the triage result plus draft reply to the CRM/helpdesk API.

## 1.7 Escalation Handling

High-risk or manually escalated tickets are sent to a dedicated escalation webhook with the triage payload and original message.

## 1.8 Observability Logging

After either terminal business path, a Code node computes and optionally sends operational metrics to an observability endpoint.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block receives support tickets from either email or webhook. It provides dual entry points so the same downstream triage logic can process messages regardless of source.

### Nodes Involved
- Email Trigger (IMAP)
- Webhook Trigger

### Node Details

#### Email Trigger (IMAP)
- **Type and technical role:** `n8n-nodes-base.emailReadImap`; polling/trigger node for incoming email via IMAP.
- **Configuration choices:** Uses default options only. No mailbox filtering or custom parsing options are explicitly set in the workflow JSON.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node; outputs to **Workflow Configuration**.
- **Version-specific requirements:** `typeVersion 2.1`.
- **Edge cases or potential failure types:**
  - IMAP authentication failure
  - SSL/TLS misconfiguration
  - mailbox access or folder permissions issues
  - unexpected email payload shape, especially if `html`, `body`, or `text` fields differ by provider
- **Sub-workflow reference:** None.

#### Webhook Trigger
- **Type and technical role:** `n8n-nodes-base.webhook`; HTTP POST entry point for external systems.
- **Configuration choices:**
  - path: `support-ticket`
  - method: `POST`
  - response mode: `lastNode`
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:** Entry node; outputs to **Workflow Configuration**.
- **Version-specific requirements:** `typeVersion 2.1`.
- **Edge cases or potential failure types:**
  - invalid or missing request body fields
  - callers waiting for response until the workflow reaches its last executed node
  - webhook path conflict with other workflows
  - large request bodies or malformed JSON
- **Sub-workflow reference:** None.

---

## 2.2 Configuration and Payload Preparation

### Overview
This block injects workflow-wide endpoint settings, extracts usable message text from potentially mixed HTML/text sources, and standardizes the ticket schema for later AI processing.

### Nodes Involved
- Workflow Configuration
- Clean HTML
- Normalize Ticket Data

### Node Details

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; adds reusable configuration fields to each item.
- **Configuration choices:**
  - `crmApiUrl`: placeholder for CRM/helpdesk API
  - `escalationWebhookUrl`: placeholder for escalation endpoint
  - `observabilityEndpoint`: placeholder for observability/logging endpoint
  - `includeOtherFields = true`, so incoming payload data is preserved
- **Key expressions or variables used:** Static placeholder strings only.
- **Input and output connections:** Receives from **Email Trigger (IMAP)** and **Webhook Trigger**; outputs to **Clean HTML**.
- **Version-specific requirements:** `typeVersion 3.4`.
- **Edge cases or potential failure types:**
  - placeholders left unmodified in production
  - later nodes referencing different field names than those set here
- **Sub-workflow reference:** None.

#### Clean HTML
- **Type and technical role:** `n8n-nodes-base.html`; extracts content from HTML/body/text input.
- **Configuration choices:**
  - operation: `extractHtmlContent`
  - source property: `{{ $json.html || $json.body || $json.text }}`
  - extraction values contain an empty object, which means the node is being used primarily to strip/extract content rather than target specific selectors
- **Key expressions or variables used:**
  - `={{ $json.html || $json.body || $json.text }}`
- **Input and output connections:** Receives from **Workflow Configuration**; outputs to **Normalize Ticket Data**.
- **Version-specific requirements:** `typeVersion 1.2`.
- **Edge cases or potential failure types:**
  - if all three fields are missing, the expression resolves to empty/undefined
  - HTML extraction behavior may vary depending on malformed markup
  - output shape may not always preserve a useful `text` field unless upstream input is well-formed
- **Sub-workflow reference:** None.

#### Normalize Ticket Data
- **Type and technical role:** `n8n-nodes-base.set`; maps varying inbound fields to a consistent support-ticket schema.
- **Configuration choices:**
  - `ticket_id = {{ $json.id || $json.messageId || $now }}`
  - `user_email = {{ $json.from || $json.email }}`
  - `original_message = {{ $json.text }}`
  - `timestamp = {{ $now }}`
  - `source_channel = {{ $json.source || "email" }}`
  - `includeOtherFields = true`
- **Key expressions or variables used:**
  - fallback ID logic uses `id`, then `messageId`, then current timestamp
  - source defaults to `"email"`
- **Input and output connections:** Receives from **Clean HTML**; outputs to **Language Detection & Translation Agent**.
- **Version-specific requirements:** `typeVersion 3.4`.
- **Edge cases or potential failure types:**
  - `original_message` depends on `$json.text`; if `Clean HTML` does not produce `text`, downstream AI can fail or classify an empty input
  - `from` may include a formatted address rather than a plain email
  - timestamp-as-ID fallback may not be stable enough for deduplication
- **Sub-workflow reference:** None.

---

## 2.3 Language Detection and Translation

### Overview
This block ensures all tickets are analyzed in English by detecting the source language and translating when required. It uses a structured output parser so downstream nodes can reliably reference named fields.

### Nodes Involved
- Language Detection & Translation Agent
- Anthropic Model - Translation
- Translation Output Parser

### Node Details

#### Language Detection & Translation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompt-driven AI agent orchestrating language detection and translation.
- **Configuration choices:**
  - input text: `{{ $json.original_message }}`
  - custom system prompt instructs the model to:
    1. detect language
    2. translate to English if needed
    3. preserve original text if already English
    4. provide confidence score
  - `promptType = define`
  - `hasOutputParser = true`
- **Key expressions or variables used:**
  - `={{ $json.original_message }}`
- **Input and output connections:**
  - main input from **Normalize Ticket Data**
  - AI language model from **Anthropic Model - Translation**
  - AI output parser from **Translation Output Parser**
  - main output to **Support Intelligence Agent**
- **Version-specific requirements:** `typeVersion 3`.
- **Edge cases or potential failure types:**
  - empty or noisy `original_message`
  - parser validation failure if the model returns non-JSON or wrong schema
  - Anthropic credential/model access issues
  - low-confidence language detection not explicitly handled by branching
- **Sub-workflow reference:** None.

#### Anthropic Model - Translation
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; chat model backing the translation agent.
- **Configuration choices:**
  - model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via `ai_languageModel` to **Language Detection & Translation Agent**.
- **Version-specific requirements:** `typeVersion 1.3`; requires Anthropic credentials in n8n.
- **Edge cases or potential failure types:**
  - credential/auth failures
  - model unavailable in workspace or region
  - token/input size issues for long emails
- **Sub-workflow reference:** None.

#### Translation Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; schema validation and structured extraction of model output.
- **Configuration choices:** Manual JSON schema with:
  - `original_language` string
  - `translated_text` string
  - `confidence` number
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via `ai_outputParser` to **Language Detection & Translation Agent**.
- **Version-specific requirements:** `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - schema mismatch
  - missing required semantics even if technically parseable
  - invalid numeric formatting for confidence
- **Sub-workflow reference:** None.

---

## 2.4 Support Intelligence Classification

### Overview
This block performs the core triage analysis. It converts the translated ticket into structured support metadata used by routing and downstream business actions.

### Nodes Involved
- Support Intelligence Agent
- Anthropic Model - Intelligence
- Intelligence Output Parser

### Node Details

#### Support Intelligence Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI triage agent for support analysis.
- **Configuration choices:**
  - input text: `{{ $json.translated_text }}`
  - system prompt requests:
    - sentiment
    - urgency
    - category
    - summary
    - churn risk
    - recommended path
  - `promptType = define`
  - `hasOutputParser = true`
- **Key expressions or variables used:**
  - `={{ $json.translated_text }}`
- **Input and output connections:**
  - main input from **Language Detection & Translation Agent**
  - AI language model from **Anthropic Model - Intelligence**
  - AI output parser from **Intelligence Output Parser**
  - main output to **Decision Router**
- **Version-specific requirements:** `typeVersion 3`.
- **Edge cases or potential failure types:**
  - poor classification on very short or ambiguous tickets
  - parser/schema mismatch
  - translated text missing due to upstream failure
- **Sub-workflow reference:** None.

#### Anthropic Model - Intelligence
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Claude Sonnet model used for triage.
- **Configuration choices:**
  - model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via `ai_languageModel` to **Support Intelligence Agent**.
- **Version-specific requirements:** `typeVersion 1.3`; requires Anthropic credentials.
- **Edge cases or potential failure types:**
  - API rate limits
  - model availability changes
  - credential problems
- **Sub-workflow reference:** None.

#### Intelligence Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces the triage JSON schema.
- **Configuration choices:** Manual JSON schema with:
  - `sentiment`: positive/neutral/negative
  - `urgency`: low/medium/high/critical
  - `category`: billing/bug/feature_request/account/technical/other
  - `summary`: string
  - `churn_risk`: number 0 to 1
  - `recommended_path`: auto_reply/escalate/engineering
  - required fields are set for all core outputs
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via `ai_outputParser` to **Support Intelligence Agent**.
- **Version-specific requirements:** `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - enum violations
  - missing required fields
  - non-numeric churn score
- **Sub-workflow reference:** None.

---

## 2.5 Decision Routing

### Overview
This block determines which operational path to execute next based on AI triage output. It explicitly supports auto-reply and escalation, while routing all unmatched cases to escalation as fallback.

### Nodes Involved
- Decision Router

### Node Details

#### Decision Router
- **Type and technical role:** `n8n-nodes-base.switch`; conditional branching node.
- **Configuration choices:**
  - Output 1 renamed to **Auto Reply**
    - condition: `{{ $json.recommended_path }} == "auto_reply"`
  - Output 2 renamed to **Escalate**
    - condition OR:
      - `{{ $json.urgency }} == "critical"`
      - `{{ $json.recommended_path }} == "escalate"`
  - fallback output renamed to **Escalate**
  - ignore case enabled
- **Key expressions or variables used:**
  - `={{ $json.recommended_path }}`
  - `={{ $json.urgency }}`
- **Input and output connections:**
  - receives from **Support Intelligence Agent**
  - output 0 to **Draft Reply Generator**
  - output 1 to **Escalate to Team**
- **Version-specific requirements:** `typeVersion 3.4`.
- **Edge cases or potential failure types:**
  - `recommended_path = engineering` is not explicitly mapped and therefore falls into escalation fallback
  - if AI output is missing or malformed, item may route to fallback unexpectedly
  - strict type validation can surface problems if fields are not strings
- **Sub-workflow reference:** None.

---

## 2.6 Auto Reply and CRM Update

### Overview
This block handles tickets suitable for direct response. It generates a concise customer-facing reply draft and pushes triage metadata plus the reply into the CRM/helpdesk system.

### Nodes Involved
- Draft Reply Generator
- Anthropic Model - Reply
- Update CRM/Helpdesk

### Node Details

#### Draft Reply Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI text generation agent for support response drafting.
- **Configuration choices:**
  - input prompt is composed from:
    - original message
    - category
    - sentiment
    - urgency
  - system prompt instructs:
    - empathetic professional tone
    - actionable solutions when possible
    - under 200 words
    - no feature/timeline promises
    - no hallucinations
    - plain reply text only
- **Key expressions or variables used:**
  - `Original message: {{ $json.original_message }}`
  - `Category: {{ $json.category }}`
  - `Sentiment: {{ $json.sentiment }}`
  - `Urgency: {{ $json.urgency }}`
- **Input and output connections:**
  - main input from **Decision Router**
  - AI language model from **Anthropic Model - Reply**
  - main output to **Update CRM/Helpdesk**
- **Version-specific requirements:** `typeVersion 3`.
- **Edge cases or potential failure types:**
  - no output parser means free-form model output; formatting may vary
  - if upstream fields are missing, prompt quality degrades
  - generated text might still need human review before sending
- **Sub-workflow reference:** None.

#### Anthropic Model - Reply
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Claude Sonnet model for reply drafting.
- **Configuration choices:**
  - model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected via `ai_languageModel` to **Draft Reply Generator**.
- **Version-specific requirements:** `typeVersion 1.3`; requires Anthropic credentials.
- **Edge cases or potential failure types:**
  - credential issues
  - rate limits
  - long prompts from large messages
- **Sub-workflow reference:** None.

#### Update CRM/Helpdesk
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends ticket analysis and draft reply to an external CRM/helpdesk API.
- **Configuration choices:**
  - method: `POST`
  - URL: `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - content type header: `application/json`
  - body fields:
    - `ticket_id`
    - `priority`
    - `tags`
    - `summary`
    - `sentiment`
    - `churn_risk`
    - `draft_reply`
    - `original_language`
- **Key expressions or variables used:**
  - pulls values from several upstream nodes using explicit node references:
    - `$('Normalize Ticket Data').item.json.ticket_id`
    - `$('Support Intelligence Agent').item.json.urgency`
    - `$('Draft Reply Generator').item.json.output`
    - `$('Language Detection & Translation Agent').item.json.original_language`
- **Input and output connections:**
  - receives from **Draft Reply Generator**
  - outputs to **Log Observability Metrics**
- **Version-specific requirements:** `typeVersion 4.3`.
- **Edge cases or potential failure types:**
  - CRM URL placeholder not replaced
  - authentication may be required but is not configured here
  - node-reference expressions can break if item linking is inconsistent across branches or batching
  - API may expect different field names/types than sent here
- **Sub-workflow reference:** None.

---

## 2.7 Escalation Handling

### Overview
This block forwards urgent or escalated tickets to a support team webhook. It sends the essential context needed for immediate manual handling.

### Nodes Involved
- Escalate to Team

### Node Details

#### Escalate to Team
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts escalated tickets to an external endpoint.
- **Configuration choices:**
  - method: `POST`
  - URL: `{{ $('Workflow Configuration').first().json.escalationWebhookUrl }}`
  - content type header: `application/json`
  - body includes:
    - `ticket_id`
    - `urgency`
    - `category`
    - `sentiment`
    - `summary`
    - `churn_risk`
    - `user_email`
    - `original_message`
- **Key expressions or variables used:** Uses current item fields directly.
- **Input and output connections:**
  - receives from **Decision Router**
  - outputs to **Log Observability Metrics**
- **Version-specific requirements:** `typeVersion 4.3`.
- **Edge cases or potential failure types:**
  - placeholder URL not replaced
  - missing auth headers if required by the webhook target
  - webhook may require a different schema
  - `user_email` may be absent or not normalized
- **Sub-workflow reference:** None.

---

## 2.8 Observability Logging

### Overview
This block computes basic workflow metrics and optionally sends them to a monitoring endpoint. It also returns a success/failure result for the logging operation itself.

### Nodes Involved
- Log Observability Metrics

### Node Details

#### Log Observability Metrics
- **Type and technical role:** `n8n-nodes-base.code`; JavaScript code node performing metric construction and optional HTTP POST.
- **Configuration choices:**
  - mode: `runOnceForEachItem`
  - builds metrics including:
    - `response_time`
    - `escalation_status`
    - `model_used`
    - `category`
    - `urgency`
    - `sentiment`
    - `churn_risk`
    - `timestamp`
  - if an observability endpoint exists, performs `$http.post(...)`
- **Key expressions or variables used in code:**
  - `$('Workflow Configuration').item.json.timestamp || new Date().toISOString()`
  - `$input.item.json.escalated || false`
  - `$('Support Intelligence Agent').item.json`
  - `$('Workflow Configuration').item.json.observability_endpoint`
- **Input and output connections:**
  - receives from **Update CRM/Helpdesk** and **Escalate to Team**
  - terminal node
- **Version-specific requirements:** `typeVersion 2`.
- **Edge cases or potential failure types:**
  - **Important field-name mismatch:** configuration node sets `observabilityEndpoint`, but code reads `observability_endpoint`; this means the endpoint will usually be treated as absent
  - **Timestamp mismatch:** code reads timestamp from **Workflow Configuration**, but that node does not define `timestamp`; the actual timestamp is created in **Normalize Ticket Data**
  - `escalated` is never explicitly set upstream, so escalation status will usually remain `false`
  - if `$http.post` is restricted or fails, code returns structured failure output
  - cross-node item references can be brittle if execution mode or item linking changes
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Email Trigger (IMAP) | n8n-nodes-base.emailReadImap | Receive support tickets from IMAP email |  | Workflow Configuration | ## Input Layer<br>Receive tickets via email or webhook |
| Webhook Trigger | n8n-nodes-base.webhook | Receive support tickets via HTTP POST |  | Workflow Configuration | ## Input Layer<br>Receive tickets via email or webhook |
| Workflow Configuration | n8n-nodes-base.set | Inject reusable endpoint configuration values | Email Trigger (IMAP), Webhook Trigger | Clean HTML | ## Configuration<br>Set CRM, escalation, and logging endpoints |
| Clean HTML | n8n-nodes-base.html | Extract usable content from HTML/body/text | Workflow Configuration | Normalize Ticket Data | ## Data Processing<br>Clean HTML and normalize ticket data |
| Normalize Ticket Data | n8n-nodes-base.set | Standardize inbound ticket fields | Clean HTML | Language Detection & Translation Agent | ## Data Processing<br>Clean HTML and normalize ticket data |
| Language Detection & Translation Agent | @n8n/n8n-nodes-langchain.agent | Detect source language and translate to English | Normalize Ticket Data | Support Intelligence Agent | ## Translation Layer<br>Detect language and translate to English |
| Anthropic Model - Translation | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for translation agent |  | Language Detection & Translation Agent | ## Translation Layer<br>Detect language and translate to English |
| Translation Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured translation output |  | Language Detection & Translation Agent | ## Translation Layer<br>Detect language and translate to English |
| Support Intelligence Agent | @n8n/n8n-nodes-langchain.agent | Classify ticket sentiment, urgency, category, and route | Language Detection & Translation Agent | Decision Router | ## AI Classification<br>Analyze sentiment, urgency, and category |
| Anthropic Model - Intelligence | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for support triage |  | Support Intelligence Agent | ## AI Classification<br>Analyze sentiment, urgency, and category |
| Intelligence Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured triage schema |  | Support Intelligence Agent | ## AI Classification<br>Analyze sentiment, urgency, and category |
| Decision Router | n8n-nodes-base.switch | Route tickets to auto-reply or escalation | Support Intelligence Agent | Draft Reply Generator, Escalate to Team | ## Decision Routing<br>Route tickets based on AI recommendation |
| Draft Reply Generator | @n8n/n8n-nodes-langchain.agent | Generate a customer-facing draft response | Decision Router | Update CRM/Helpdesk | ## Auto Reply Flow<br>Generate AI reply and update CRM |
| Anthropic Model - Reply | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for reply generation |  | Draft Reply Generator | ## Auto Reply Flow<br>Generate AI reply and update CRM |
| Update CRM/Helpdesk | n8n-nodes-base.httpRequest | Push analysis and draft reply to CRM/helpdesk | Draft Reply Generator | Log Observability Metrics | ## Auto Reply Flow<br>Generate AI reply and update CRM |
| Escalate to Team | n8n-nodes-base.httpRequest | Forward urgent tickets to support team webhook | Decision Router | Log Observability Metrics | ## Escalation Flow<br>Send high-risk tickets to support team |
| Log Observability Metrics | n8n-nodes-base.code | Build and optionally send monitoring metrics | Update CRM/Helpdesk, Escalate to Team |  | ## Observability<br>Log metrics for monitoring and insights |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name matching the intended support-triage purpose.

2. **Add the first trigger node: `Email Trigger (IMAP)`**
   - Node type: **Email Trigger / IMAP**
   - Leave default options unless you want a specific mailbox, folder, or filters.
   - Configure IMAP credentials:
     - host
     - port
     - username
     - password or app password
     - SSL/TLS as required by your provider

3. **Add the second trigger node: `Webhook Trigger`**
   - Node type: **Webhook**
   - HTTP Method: `POST`
   - Path: `support-ticket`
   - Response Mode: `Last Node`
   - This creates the second entry point for non-email tickets.

4. **Add `Workflow Configuration`**
   - Node type: **Set**
   - Enable keeping existing fields: `Include Other Fields = true`
   - Add three string fields:
     - `crmApiUrl`
     - `escalationWebhookUrl`
     - `observabilityEndpoint`
   - Populate them with your real endpoints or placeholders during setup.

5. **Connect both triggers to `Workflow Configuration`**
   - `Email Trigger (IMAP) -> Workflow Configuration`
   - `Webhook Trigger -> Workflow Configuration`

6. **Add `Clean HTML`**
   - Node type: **HTML**
   - Operation: `Extract HTML Content`
   - Data Property Name:
     - `{{ $json.html || $json.body || $json.text }}`
   - Leave extraction values minimal as in the source workflow.

7. **Connect `Workflow Configuration -> Clean HTML`**

8. **Add `Normalize Ticket Data`**
   - Node type: **Set**
   - `Include Other Fields = true`
   - Add these fields:
     - `ticket_id` = `{{ $json.id || $json.messageId || $now }}`
     - `user_email` = `{{ $json.from || $json.email }}`
     - `original_message` = `{{ $json.text }}`
     - `timestamp` = `{{ $now }}`
     - `source_channel` = `{{ $json.source || "email" }}`
   - This node is critical because later nodes assume these field names exist.

9. **Connect `Clean HTML -> Normalize Ticket Data`**

10. **Add `Language Detection & Translation Agent`**
    - Node type: **AI Agent** from the LangChain n8n package
    - Prompt type: `Define`
    - Text input:
      - `{{ $json.original_message }}`
    - Enable structured output / output parser
    - System message:
      - Instruct the model to detect language, translate non-English to English, keep English unchanged, and return confidence.

11. **Add `Anthropic Model - Translation`**
    - Node type: **Anthropic Chat Model**
    - Select Anthropic credentials
    - Model: `Claude Sonnet 4.5` or the exact listed model if available: `claude-sonnet-4-5-20250929`

12. **Add `Translation Output Parser`**
    - Node type: **Structured Output Parser**
    - Schema type: `Manual`
    - Define a schema with:
      - `original_language` as string
      - `translated_text` as string
      - `confidence` as number

13. **Wire the translation block**
    - Main: `Normalize Ticket Data -> Language Detection & Translation Agent`
    - AI language model: `Anthropic Model - Translation -> Language Detection & Translation Agent`
    - AI output parser: `Translation Output Parser -> Language Detection & Translation Agent`

14. **Add `Support Intelligence Agent`**
    - Node type: **AI Agent**
    - Prompt type: `Define`
    - Text input:
      - `{{ $json.translated_text }}`
    - Enable output parser
    - System message should request:
      - sentiment
      - urgency
      - category
      - summary
      - churn risk
      - recommended path

15. **Add `Anthropic Model - Intelligence`**
    - Node type: **Anthropic Chat Model**
    - Reuse Anthropic credentials
    - Use the same Claude Sonnet model

16. **Add `Intelligence Output Parser`**
    - Node type: **Structured Output Parser**
    - Schema type: `Manual`
    - Define:
      - `sentiment`: positive, neutral, negative
      - `urgency`: low, medium, high, critical
      - `category`: billing, bug, feature_request, account, technical, other
      - `summary`: string
      - `churn_risk`: number between 0 and 1
      - `recommended_path`: auto_reply, escalate, engineering
    - Mark these fields as required.

17. **Wire the intelligence block**
    - Main: `Language Detection & Translation Agent -> Support Intelligence Agent`
    - AI language model: `Anthropic Model - Intelligence -> Support Intelligence Agent`
    - AI output parser: `Intelligence Output Parser -> Support Intelligence Agent`

18. **Add `Decision Router`**
    - Node type: **Switch**
    - Create output rule 1 named `Auto Reply`
      - Condition: `{{ $json.recommended_path }}` equals `auto_reply`
    - Create output rule 2 named `Escalate`
      - Condition group with OR:
        - `{{ $json.urgency }}` equals `critical`
        - `{{ $json.recommended_path }}` equals `escalate`
    - Enable fallback output and rename it to `Escalate`
    - Optionally enable ignore-case matching

19. **Connect `Support Intelligence Agent -> Decision Router`**

20. **Add `Draft Reply Generator`**
    - Node type: **AI Agent**
    - Prompt type: `Define`
    - Use a composed text prompt such as:
      - Original message
      - Category
      - Sentiment
      - Urgency
    - System prompt should require:
      - empathetic tone
      - under 200 words
      - no hallucinations
      - no unapproved promises
      - plain reply text only
    - No output parser is required for this version.

21. **Add `Anthropic Model - Reply`**
    - Node type: **Anthropic Chat Model**
    - Reuse Anthropic credentials
    - Use the same Sonnet model

22. **Wire the auto-reply generation path**
    - Main: `Decision Router (Auto Reply output) -> Draft Reply Generator`
    - AI language model: `Anthropic Model - Reply -> Draft Reply Generator`

23. **Add `Update CRM/Helpdesk`**
    - Node type: **HTTP Request**
    - Method: `POST`
    - URL:
      - `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
    - Send Headers: enabled
    - Add header:
      - `Content-Type: application/json`
    - Send Body: enabled
    - Add body parameters:
      - `ticket_id` = `{{ $('Normalize Ticket Data').item.json.ticket_id }}`
      - `priority` = `{{ $('Support Intelligence Agent').item.json.urgency }}`
      - `tags` = `{{ $('Support Intelligence Agent').item.json.category }}`
      - `summary` = `{{ $('Support Intelligence Agent').item.json.summary }}`
      - `sentiment` = `{{ $('Support Intelligence Agent').item.json.sentiment }}`
      - `churn_risk` = `{{ $('Support Intelligence Agent').item.json.churn_risk }}`
      - `draft_reply` = `{{ $('Draft Reply Generator').item.json.output }}`
      - `original_language` = `{{ $('Language Detection & Translation Agent').item.json.original_language }}`
    - If your CRM needs authentication, add:
      - bearer token header, or
      - OAuth2 credentials, or
      - API key header/query parameter

24. **Connect `Draft Reply Generator -> Update CRM/Helpdesk`**

25. **Add `Escalate to Team`**
    - Node type: **HTTP Request**
    - Method: `POST`
    - URL:
      - `{{ $('Workflow Configuration').first().json.escalationWebhookUrl }}`
    - Send Headers: enabled
    - Add header:
      - `Content-Type: application/json`
    - Send Body: enabled
    - Add body parameters:
      - `ticket_id` = `{{ $json.ticket_id }}`
      - `urgency` = `{{ $json.urgency }}`
      - `category` = `{{ $json.category }}`
      - `sentiment` = `{{ $json.sentiment }}`
      - `summary` = `{{ $json.summary }}`
      - `churn_risk` = `{{ $json.churn_risk }}`
      - `user_email` = `{{ $json.user_email }}`
      - `original_message` = `{{ $json.original_message }}`
    - Add authentication if the escalation service requires it.

26. **Connect `Decision Router (Escalate output) -> Escalate to Team`**

27. **Add `Log Observability Metrics`**
    - Node type: **Code**
    - Mode: `Run Once for Each Item`
    - Paste logic that:
      - computes response time
      - determines escalation state
      - extracts triage metadata
      - optionally POSTs metrics to an observability endpoint
      - returns structured success or failure data

28. **Connect both business paths to the logging node**
    - `Update CRM/Helpdesk -> Log Observability Metrics`
    - `Escalate to Team -> Log Observability Metrics`

29. **Correct the configuration mismatch before production**
    - In the provided workflow, the Code node reads:
      - `observability_endpoint`
    - But the Set node creates:
      - `observabilityEndpoint`
    - Standardize on one field name. Best fix:
      - either rename the Set field to `observability_endpoint`
      - or update the Code node to read `observabilityEndpoint`

30. **Correct the timestamp source**
    - The Code node tries to read a timestamp from `Workflow Configuration`
    - But the actual `timestamp` is created in `Normalize Ticket Data`
    - Update the code to use:
      - `$('Normalize Ticket Data').item.json.timestamp`

31. **If you need reliable escalation metrics, add an explicit flag**
    - Before or after `Escalate to Team`, add a Set node that defines `escalated = true`
    - Before or after the CRM path, optionally define `escalated = false`
    - This will make observability metrics accurate

32. **Test the email path**
    - Send a sample email with HTML and plain text
    - Verify that `original_message`, translation output, classification, routing, and CRM/escalation behavior all work

33. **Test the webhook path**
    - Send a POST request to `/support-ticket`
    - Include fields such as `id`, `email`, `text`, and `source`
    - Confirm normalization and routing are identical to the email path

34. **Validate model access**
    - Ensure Anthropic credentials are available and the selected Claude Sonnet model is enabled in your n8n environment
    - If not, select another supported Anthropic model and retest parser compatibility

35. **Activate the workflow**
    - After successful tests for both entry points, enable the workflow in production.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates customer support ticket processing using AI-powered analysis and intelligent routing. Incoming messages from email or webhook are cleaned, normalized, and translated into English if needed. An AI agent then analyzes each ticket to determine sentiment, urgency, category, and churn risk. All actions are logged with observability metrics for monitoring performance and response quality. | General workflow purpose |
| Configure IMAP or webhook for incoming tickets | Setup guidance |
| Add AI model credentials (Anthropic/OpenAI) | Setup guidance |
| Set CRM or helpdesk API endpoint | Setup guidance |
| Configure escalation webhook for alerts | Setup guidance |
| Set observability/logging endpoint (optional) | Setup guidance |

## Additional implementation notes
- There are **two explicit entry points**: IMAP email and webhook.
- There are **no sub-workflows** and no Execute Workflow nodes in this workflow.
- The workflow uses **three Anthropic-backed AI agent stages**:
  - translation
  - support classification
  - reply generation
- The `engineering` value from `recommended_path` is not handled as a separate branch; it will fall through to escalation.
- The workflow currently appears intended more for **triage and draft generation** than direct message sending, because no outbound email/send-message node exists.