Triage and reply to multilingual support tickets with Anthropic Claude

https://n8nworkflows.xyz/workflows/triage-and-reply-to-multilingual-support-tickets-with-anthropic-claude-13940


# Triage and reply to multilingual support tickets with Anthropic Claude

# 1. Workflow Overview

This workflow ingests support tickets from either an email inbox or an HTTP webhook, standardizes the incoming payload, translates non-English messages into English, classifies the ticket with Anthropic Claude, then routes the ticket into either an auto-reply path or an escalation path. It also updates an external CRM/helpdesk system and attempts to log observability metrics.

Typical use cases:
- Centralized multilingual support intake
- Automated first-pass triage for inbound support traffic
- Auto-drafting responses for lower-risk tickets
- Escalating urgent or high-risk issues to a support team
- Sending structured ticket intelligence to downstream systems

## 1.1 Input Reception

Two entry points receive tickets:
- **Email Trigger (IMAP)** for inbox polling
- **Webhook Trigger** for POSTed ticket payloads

Both feed the same downstream pipeline.

## 1.2 Configuration and Payload Preparation

The workflow injects runtime configuration values such as the CRM API URL, escalation webhook URL, and observability endpoint, then cleans message HTML and normalizes ticket fields into a consistent schema.

## 1.3 Multilingual Processing

An Anthropic-powered AI agent detects the source language and translates the message to English when needed. Structured output ensures downstream nodes receive predictable fields.

## 1.4 Ticket Intelligence and Triage

A second Anthropic-powered AI agent classifies the translated message for sentiment, urgency, category, summary, churn risk, and recommended path. A switch node uses this output to decide whether to auto-reply or escalate.

## 1.5 Auto-Reply Path

If the recommended path is `auto_reply`, the workflow generates a draft response using ticket context and sends the enriched data to a CRM/helpdesk endpoint.

## 1.6 Escalation Path

If the ticket is urgent or explicitly marked for escalation, the workflow sends a structured payload to an escalation webhook for human team handling.

## 1.7 Observability

After either the CRM update or escalation call, a Code node computes and optionally posts runtime metrics. This block is intended for monitoring, but it contains configuration mismatches that should be fixed before production use.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block accepts inbound support tickets from two separate channels and funnels them into one shared processing pipeline. It allows the same automation logic to be reused regardless of whether the source is email or an external system.

### Nodes Involved
- Email Trigger (IMAP)
- Webhook Trigger

### Node Details

#### Email Trigger (IMAP)
- **Type and technical role:** `n8n-nodes-base.emailReadImap` trigger node; polls or listens for incoming emails from an IMAP mailbox.
- **Configuration choices:** Minimal/default configuration in the JSON; actual mailbox behavior depends on credentials and any runtime IMAP settings defined in the node UI.
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Entry point node with output to **Workflow Configuration**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - IMAP authentication failure
  - TLS or server connectivity issues
  - Unexpected email structure, especially plain-text vs HTML-only emails
  - Missing `from`, `text`, `html`, or `messageId` fields depending on provider behavior
- **Sub-workflow reference:** None.

#### Webhook Trigger
- **Type and technical role:** `n8n-nodes-base.webhook`; accepts HTTP POST requests for externally submitted support tickets.
- **Configuration choices:**
  - Path: `support-ticket`
  - Method: `POST`
  - Response mode: `lastNode`, meaning the webhook response comes from the final node in the executed branch
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Entry point node with output to **Workflow Configuration**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Caller sends malformed JSON or missing expected fields
  - Timeout if downstream AI or HTTP calls are slow
  - Response shape differs depending on which branch completes
- **Sub-workflow reference:** None.

---

## 2.2 Configuration and Payload Preparation

### Overview
This block centralizes external endpoint settings and transforms raw inbound ticket data into a normalized schema suitable for AI processing. It also strips HTML to produce cleaner message text.

### Nodes Involved
- Workflow Configuration
- Clean HTML
- Normalize Ticket Data

### Node Details

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; injects static configuration values into each item.
- **Configuration choices:**
  - Adds:
    - `crmApiUrl`
    - `escalationWebhookUrl`
    - `observabilityEndpoint`
  - Keeps original incoming fields via `includeOtherFields: true`
- **Key expressions or variables used:** Placeholder strings for environment-specific values.
- **Input and output connections:** Receives from **Email Trigger (IMAP)** and **Webhook Trigger**; outputs to **Clean HTML**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged, causing downstream HTTP requests to fail
  - Configuration naming inconsistency later in the workflow: downstream observability code expects `observability_endpoint`, but this node creates `observabilityEndpoint`
- **Sub-workflow reference:** None.

#### Clean HTML
- **Type and technical role:** `n8n-nodes-base.html`; extracts readable content from HTML or body text.
- **Configuration choices:**
  - Operation: `extractHtmlContent`
  - Source data property: `{{ $json.html || $json.body || $json.text }}`
  - Extraction config is effectively generic/minimal
- **Key expressions or variables used:**
  - `{{ $json.html || $json.body || $json.text }}`
- **Input and output connections:** Receives from **Workflow Configuration**; outputs to **Normalize Ticket Data**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - If all of `html`, `body`, and `text` are empty, extraction may yield empty output
  - Webhook payloads with non-string message bodies may break extraction
  - HTML extraction result may not always map cleanly back to `text`, depending on node behavior
- **Sub-workflow reference:** None.

#### Normalize Ticket Data
- **Type and technical role:** `n8n-nodes-base.set`; creates a consistent ticket schema regardless of source.
- **Configuration choices:**
  - `ticket_id = {{ $json.id || $json.messageId || $now }}`
  - `user_email = {{ $json.from || $json.email }}`
  - `original_message = {{ $json.text }}`
  - `timestamp = {{ $now }}`
  - `source_channel = {{ $json.source || "email" }}`
  - Preserves original fields
- **Key expressions or variables used:**
  - `{{ $json.id || $json.messageId || $now }}`
  - `{{ $json.from || $json.email }}`
  - `{{ $json.text }}`
  - `{{ $now }}`
  - `{{ $json.source || "email" }}`
- **Input and output connections:** Receives from **Clean HTML**; outputs to **Language Detection & Translation Agent**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `original_message` depends on `$json.text`; if the HTML node does not populate `text`, this may be blank
  - `from` from IMAP may include display name plus address rather than a clean email address
  - Webhook tickets without `id` or email fields degrade into fallback behavior
- **Sub-workflow reference:** None.

---

## 2.3 Multilingual Processing

### Overview
This block determines the message language and translates the content into English for consistent downstream triage. Structured parsing enforces a predictable schema from the LLM output.

### Nodes Involved
- Language Detection & Translation Agent
- Anthropic Model - Translation
- Translation Output Parser

### Node Details

#### Language Detection & Translation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates an LLM prompt for language detection and translation.
- **Configuration choices:**
  - Input text: `{{ $json.original_message }}`
  - Prompt instructs the model to:
    1. Detect language
    2. Translate to English if needed
    3. Return original if already English
    4. Provide confidence score
  - Uses a structured output parser
- **Key expressions or variables used:**
  - `{{ $json.original_message }}`
- **Input and output connections:**
  - Main input from **Normalize Ticket Data**
  - AI language model input from **Anthropic Model - Translation**
  - AI output parser input from **Translation Output Parser**
  - Main output to **Support Intelligence Agent**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Empty or very short messages may yield poor language confidence
  - Mixed-language messages may produce unstable output
  - LLM may still fail schema compliance despite parser
  - Anthropic credential or rate-limit errors
- **Sub-workflow reference:** None.

#### Anthropic Model - Translation
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the Claude model used by the translation agent.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to **Language Detection & Translation Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Invalid Anthropic credentials
  - Model access restrictions
  - Token limits on unusually large messages
- **Sub-workflow reference:** None.

#### Translation Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates/structures the translation agent output.
- **Configuration choices:**
  - Manual JSON schema with:
    - `original_language` as string
    - `translated_text` as string
    - `confidence` as number
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_outputParser` to **Language Detection & Translation Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Schema mismatch if the model returns non-JSON or malformed JSON
  - `original_language` is described as ISO code but not validated by regex or enum
- **Sub-workflow reference:** None.

---

## 2.4 Ticket Intelligence and Triage

### Overview
This block analyzes the English-normalized ticket and determines customer sentiment, urgency, issue category, churn risk, summary, and routing recommendation. A switch then directs the item into either the reply branch or escalation branch.

### Nodes Involved
- Support Intelligence Agent
- Anthropic Model - Intelligence
- Intelligence Output Parser
- Decision Router

### Node Details

#### Support Intelligence Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; performs support-ticket triage with an LLM.
- **Configuration choices:**
  - Input text: `{{ $json.translated_text }}`
  - Prompt requests:
    - `sentiment`
    - `urgency`
    - `category`
    - `summary`
    - `churn_risk`
    - `recommended_path`
  - Structured output enforced via parser
- **Key expressions or variables used:**
  - `{{ $json.translated_text }}`
- **Input and output connections:**
  - Main input from **Language Detection & Translation Agent**
  - AI language model from **Anthropic Model - Intelligence**
  - AI output parser from **Intelligence Output Parser**
  - Main output to **Decision Router**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If translation output is missing `translated_text`, the agent receives empty input
  - Classification subjectivity may produce inconsistent category/routing on ambiguous tickets
  - LLM schema failures or Anthropic API issues
- **Sub-workflow reference:** None.

#### Anthropic Model - Intelligence
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Claude model backend for support classification.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to **Support Intelligence Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Credential or quota problems
  - Large input causing model truncation or latency
- **Sub-workflow reference:** None.

#### Intelligence Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates triage output against a manual schema.
- **Configuration choices:**
  - Schema includes:
    - `sentiment`: positive, neutral, negative
    - `urgency`: low, medium, high, critical
    - `category`: billing, bug, feature_request, account, technical, other
    - `summary`: string
    - `churn_risk`: number 0–1
    - `recommended_path`: auto_reply, escalate, engineering
  - Required fields enforced
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_outputParser` to **Support Intelligence Agent**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - The schema allows `engineering`, but downstream routing does not create a dedicated engineering branch; such tickets fall back to escalation
- **Sub-workflow reference:** None.

#### Decision Router
- **Type and technical role:** `n8n-nodes-base.switch`; routes tickets based on urgency and recommended action.
- **Configuration choices:**
  - Output 1: **Auto Reply**
    - condition: `recommended_path == "auto_reply"`
  - Output 2: **Escalate**
    - conditions:
      - `urgency == "critical"` OR
      - `recommended_path == "escalate"`
  - Fallback output renamed to **Escalate**
  - Ignore case enabled
- **Key expressions or variables used:**
  - `{{ $json.recommended_path }}`
  - `{{ $json.urgency }}`
- **Input and output connections:** Receives from **Support Intelligence Agent**; outputs to **Draft Reply Generator** or **Escalate to Team**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `recommended_path = "engineering"` is not handled explicitly and therefore falls into fallback escalation
  - Missing triage fields can send items to fallback unexpectedly
- **Sub-workflow reference:** None.

---

## 2.5 Auto-Reply Path

### Overview
This block drafts a response for lower-risk tickets and pushes the enriched ticket record to the CRM/helpdesk platform. It depends heavily on upstream AI outputs and cross-node expressions.

### Nodes Involved
- Draft Reply Generator
- Anthropic Model - Reply
- Update CRM/Helpdesk

### Node Details

#### Draft Reply Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; generates a concise support reply.
- **Configuration choices:**
  - Input prompt includes:
    - Original message
    - Category
    - Sentiment
    - Urgency
  - System prompt constraints:
    - empathetic and professional
    - actionable where possible
    - under 200 words
    - no unsupported promises
    - no hallucinations
    - acknowledge unresolved issues
  - Returns plain text only
- **Key expressions or variables used:**
  - `Original message: {{ $json.original_message }}`
  - `Category: {{ $json.category }}`
  - `Sentiment: {{ $json.sentiment }}`
  - `Urgency: {{ $json.urgency }}`
- **Input and output connections:**
  - Main input from **Decision Router**
  - AI language model from **Anthropic Model - Reply**
  - Main output to **Update CRM/Helpdesk**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If the input item no longer contains `original_message` or classification fields due to merge behavior, the draft may be poor or blank
  - No structured parser is used, so output format relies entirely on prompt adherence
- **Sub-workflow reference:** None.

#### Anthropic Model - Reply
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Claude backend for reply drafting.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to **Draft Reply Generator**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Auth/rate-limit/model-access failures
  - Long customer messages may impact response quality or latency
- **Sub-workflow reference:** None.

#### Update CRM/Helpdesk
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends enriched ticket data to an external CRM/helpdesk API.
- **Configuration choices:**
  - Method: `POST`
  - URL from `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - Header: `Content-Type: application/json`
  - Body fields:
    - `ticket_id`
    - `priority`
    - `tags`
    - `summary`
    - `sentiment`
    - `churn_risk`
    - `draft_reply`
    - `original_language`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - `{{ $('Normalize Ticket Data').item.json.ticket_id }}`
  - `{{ $('Support Intelligence Agent').item.json.urgency }}`
  - `{{ $('Support Intelligence Agent').item.json.category }}`
  - `{{ $('Support Intelligence Agent').item.json.summary }}`
  - `{{ $('Support Intelligence Agent').item.json.sentiment }}`
  - `{{ $('Support Intelligence Agent').item.json.churn_risk }}`
  - `{{ $('Draft Reply Generator').item.json.output }}`
  - `{{ $('Language Detection & Translation Agent').item.json.original_language }}`
- **Input and output connections:** Receives from **Draft Reply Generator**; outputs to **Log Observability Metrics**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Placeholder CRM URL not replaced
  - Remote API auth not configured in this node
  - Body parameter format may not match expected API schema
  - Cross-node expression resolution may fail if item linking breaks in multi-item executions
- **Sub-workflow reference:** None.

---

## 2.6 Escalation Path

### Overview
This block posts urgent or escalation-worthy tickets to an external team webhook. It is the fallback path for critical tickets and any routing outputs not explicitly handled by the auto-reply branch.

### Nodes Involved
- Escalate to Team

### Node Details

#### Escalate to Team
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts escalation data to an external endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL from `{{ $('Workflow Configuration').first().json.escalationWebhookUrl }}`
  - Header: `Content-Type: application/json`
  - Body fields:
    - `ticket_id`
    - `urgency`
    - `category`
    - `sentiment`
    - `summary`
    - `churn_risk`
    - `user_email`
    - `original_message`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.escalationWebhookUrl }}`
  - `{{ $json.ticket_id }}`
  - `{{ $json.urgency }}`
  - `{{ $json.category }}`
  - `{{ $json.sentiment }}`
  - `{{ $json.summary }}`
  - `{{ $json.churn_risk }}`
  - `{{ $json.user_email }}`
  - `{{ $json.original_message }}`
- **Input and output connections:** Receives from **Decision Router**; outputs to **Log Observability Metrics**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Placeholder escalation URL not replaced
  - Downstream webhook may expect authentication or a different payload shape
  - No explicit `escalated` flag is added, even though observability code later tries to read one
- **Sub-workflow reference:** None.

---

## 2.7 Observability

### Overview
This block computes response-time and classification metadata and optionally posts it to an observability endpoint. As currently implemented, it contains field-name mismatches and timing assumptions that limit its reliability.

### Nodes Involved
- Log Observability Metrics

### Node Details

#### Log Observability Metrics
- **Type and technical role:** `n8n-nodes-base.code`; computes metrics and optionally sends them using `$http.post`.
- **Configuration choices:**
  - Runs once for each item
  - Calculates:
    - `response_time`
    - `escalation_status`
    - `model_used`
    - `category`
    - `urgency`
    - `sentiment`
    - `churn_risk`
    - `timestamp`
  - Attempts to post to endpoint if configured
- **Key expressions or variables used inside code:**
  - `$('Workflow Configuration').item.json.timestamp || new Date().toISOString()`
  - `$input.item.json.escalated || false`
  - `$('Support Intelligence Agent').item.json`
  - `$('Workflow Configuration').item.json.observability_endpoint`
- **Input and output connections:** Receives from **Update CRM/Helpdesk** and **Escalate to Team**; terminal node for both branches.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - **Bug:** `Workflow Configuration` does not set `timestamp`; actual timestamp is created in **Normalize Ticket Data**
  - **Bug:** `Workflow Configuration` sets `observabilityEndpoint`, but code reads `observability_endpoint`
  - **Bug:** `escalated` is never set upstream, so `escalation_status` is always false unless the input payload already had that field
  - If `$http.post` is unsupported or restricted in the runtime, posting fails
  - Metrics may represent only branch completion time, not end-to-end SLA
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Email Trigger (IMAP) | n8n-nodes-base.emailReadImap | Receives support tickets from an IMAP inbox |  | Workflow Configuration | Receives support tickets from two sources:<br>• Email inbox (IMAP trigger)<br>• Webhook endpoint<br><br>Both sources feed messages into the same processing pipeline. |
| Webhook Trigger | n8n-nodes-base.webhook | Receives support tickets via HTTP POST |  | Workflow Configuration | Receives support tickets from two sources:<br>• Email inbox (IMAP trigger)<br>• Webhook endpoint<br><br>Both sources feed messages into the same processing pipeline. |
| Workflow Configuration | n8n-nodes-base.set | Injects shared endpoint configuration into each item | Email Trigger (IMAP), Webhook Trigger | Clean HTML | Cleans HTML content and normalizes ticket fields such as:<br>ticket ID, email, message body, timestamp, and source channel. |
| Clean HTML | n8n-nodes-base.html | Extracts readable text from HTML/body/text input | Workflow Configuration | Normalize Ticket Data | Cleans HTML content and normalizes ticket fields such as:<br>ticket ID, email, message body, timestamp, and source channel. |
| Normalize Ticket Data | n8n-nodes-base.set | Maps source-specific fields into a normalized ticket schema | Clean HTML | Language Detection & Translation Agent | Cleans HTML content and normalizes ticket fields such as:<br>ticket ID, email, message body, timestamp, and source channel. |
| Language Detection & Translation Agent | @n8n/n8n-nodes-langchain.agent | Detects language and translates non-English text to English | Normalize Ticket Data | Support Intelligence Agent | ## AI AGENT for multi Language support<br>Detects the language of the support message and translates it to English when required so downstream AI analysis works consistently. |
| Anthropic Model - Translation | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for translation agent |  | Language Detection & Translation Agent | ## AI AGENT for multi Language support<br>Detects the language of the support message and translates it to English when required so downstream AI analysis works consistently. |
| Translation Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON for translation output |  | Language Detection & Translation Agent | ## AI AGENT for multi Language support<br>Detects the language of the support message and translates it to English when required so downstream AI analysis works consistently. |
| Support Intelligence Agent | @n8n/n8n-nodes-langchain.agent | Classifies ticket sentiment, urgency, category, churn risk, and route | Language Detection & Translation Agent | Decision Router | ## AI AGENT analyzes tickets<br>AI analyzes sentiment, urgency, category, churn risk, and generates a short summary to guide ticket routing decisions. |
| Anthropic Model - Intelligence | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for triage analysis |  | Support Intelligence Agent | ## AI AGENT analyzes tickets<br>AI analyzes sentiment, urgency, category, churn risk, and generates a short summary to guide ticket routing decisions. |
| Intelligence Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON for ticket intelligence output |  | Support Intelligence Agent | ## AI AGENT analyzes tickets<br>AI analyzes sentiment, urgency, category, churn risk, and generates a short summary to guide ticket routing decisions. |
| Decision Router | n8n-nodes-base.switch | Routes tickets to auto-reply or escalation | Support Intelligence Agent | Draft Reply Generator, Escalate to Team | Decision router sends tickets either to:<br>• Auto reply generation<br>• Escalation webhook for urgent issues |
| Draft Reply Generator | @n8n/n8n-nodes-langchain.agent | Drafts a support response for lower-risk tickets | Decision Router | Update CRM/Helpdesk | ## generate reply with context<br>Category, Sentiment:, Urgency. |
| Anthropic Model - Reply | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for reply generation |  | Draft Reply Generator | ## generate reply with context<br>Category, Sentiment:, Urgency. |
| Update CRM/Helpdesk | n8n-nodes-base.httpRequest | Sends enriched ticket data and draft reply to CRM/helpdesk | Draft Reply Generator | Log Observability Metrics | Updates the CRM/helpdesk system with ticket insights and logs workflow metrics such as response time, sentiment, and urgency. |
| Escalate to Team | n8n-nodes-base.httpRequest | Sends urgent ticket payload to an escalation webhook | Decision Router | Log Observability Metrics | ## Escalate to Team |
| Log Observability Metrics | n8n-nodes-base.code | Computes and optionally sends monitoring metrics | Update CRM/Helpdesk, Escalate to Team |  | ## ILog Observability Metrics |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the first trigger node: Email Trigger (IMAP)**
   - Add node type: **Email Trigger (IMAP)**.
   - Configure IMAP credentials for the mailbox that receives support requests.
   - Keep default options unless you need mailbox/folder filtering.
   - Name it **Email Trigger (IMAP)**.

2. **Create the second trigger node: Webhook Trigger**
   - Add node type: **Webhook**.
   - Set:
     - **HTTP Method:** `POST`
     - **Path:** `support-ticket`
     - **Response Mode:** `Last Node`
   - Name it **Webhook Trigger**.
   - If this will be called externally, activate the workflow and use the production webhook URL.

3. **Create the configuration node**
   - Add node type: **Set**.
   - Name it **Workflow Configuration**.
   - Enable keeping incoming fields.
   - Add string fields:
     - `crmApiUrl`
     - `escalationWebhookUrl`
     - `observabilityEndpoint`
   - Fill them with your real endpoints.
   - Connect both triggers to this node.

4. **Create the HTML cleanup node**
   - Add node type: **HTML**.
   - Name it **Clean HTML**.
   - Set **Operation** to **Extract HTML Content**.
   - Set the input property expression to:
     - `{{ $json.html || $json.body || $json.text }}`
   - Keep extraction values minimal/default.
   - Connect **Workflow Configuration** → **Clean HTML**.

5. **Create the normalization node**
   - Add node type: **Set**.
   - Name it **Normalize Ticket Data**.
   - Keep incoming fields.
   - Add the following fields:
     - `ticket_id` = `{{ $json.id || $json.messageId || $now }}`
     - `user_email` = `{{ $json.from || $json.email }}`
     - `original_message` = `{{ $json.text }}`
     - `timestamp` = `{{ $now }}`
     - `source_channel` = `{{ $json.source || "email" }}`
   - Connect **Clean HTML** → **Normalize Ticket Data**.

6. **Create the translation model node**
   - Add node type: **Anthropic Chat Model** from the LangChain-compatible nodes.
   - Name it **Anthropic Model - Translation**.
   - Configure **Anthropic API credentials**.
   - Select model: **Claude Sonnet 4.5** / `claude-sonnet-4-5-20250929`.

7. **Create the translation parser node**
   - Add node type: **Structured Output Parser**.
   - Name it **Translation Output Parser**.
   - Use manual schema with:
     - `original_language` as string
     - `translated_text` as string
     - `confidence` as number

8. **Create the translation agent**
   - Add node type: **AI Agent**.
   - Name it **Language Detection & Translation Agent**.
   - Set text input to:
     - `{{ $json.original_message }}`
   - Set prompt mode to define your own prompt.
   - Use this system behavior:
     - detect language
     - translate to English if not English
     - preserve meaning and tone
     - return original if already English
     - output confidence score
     - return exact JSON matching the parser schema
   - Enable structured output / output parser.
   - Connect:
     - **Normalize Ticket Data** → **Language Detection & Translation Agent**
     - **Anthropic Model - Translation** → AI language model port of the agent
     - **Translation Output Parser** → AI output parser port of the agent

9. **Create the intelligence model node**
   - Add another **Anthropic Chat Model** node.
   - Name it **Anthropic Model - Intelligence**.
   - Use the same Anthropic credentials and Claude Sonnet 4.5 model.

10. **Create the intelligence parser**
    - Add another **Structured Output Parser**.
    - Name it **Intelligence Output Parser**.
    - Define a manual schema with:
      - `sentiment`: `positive | neutral | negative`
      - `urgency`: `low | medium | high | critical`
      - `category`: `billing | bug | feature_request | account | technical | other`
      - `summary`: string
      - `churn_risk`: number between 0 and 1
      - `recommended_path`: `auto_reply | escalate | engineering`
    - Mark all key fields as required.

11. **Create the support intelligence agent**
    - Add node type: **AI Agent**.
    - Name it **Support Intelligence Agent**.
    - Set text input to:
      - `{{ $json.translated_text }}`
    - Add a system prompt instructing the model to classify:
      - sentiment
      - urgency
      - category
      - summary
      - churn risk
      - recommended path
    - Require exact structured JSON matching the parser.
    - Connect:
      - **Language Detection & Translation Agent** → **Support Intelligence Agent**
      - **Anthropic Model - Intelligence** → AI language model port
      - **Intelligence Output Parser** → AI output parser port

12. **Create the decision router**
    - Add node type: **Switch**.
    - Name it **Decision Router**.
    - Add output rule **Auto Reply**:
      - `{{ $json.recommended_path }}` equals `auto_reply`
    - Add output rule **Escalate**:
      - `{{ $json.urgency }}` equals `critical`
      - OR `{{ $json.recommended_path }}` equals `escalate`
    - Enable fallback output and rename it **Escalate**.
    - Connect **Support Intelligence Agent** → **Decision Router**.

13. **Create the reply model node**
    - Add another **Anthropic Chat Model** node.
    - Name it **Anthropic Model - Reply**.
    - Use the same Anthropic credentials and model.

14. **Create the draft reply generator**
    - Add node type: **AI Agent**.
    - Name it **Draft Reply Generator**.
    - Set the text/prompt input to include:
      - original message
      - category
      - sentiment
      - urgency
    - Example expression:
      - `Original message: {{ $json.original_message }}`
      - `Category: {{ $json.category }}`
      - `Sentiment: {{ $json.sentiment }}`
      - `Urgency: {{ $json.urgency }}`
    - Add a system prompt telling the model to:
      - be empathetic and professional
      - keep under 200 words
      - avoid hallucinations
      - avoid promising unsupported features/timelines
      - return only the reply text
    - Connect:
      - **Decision Router** auto-reply output → **Draft Reply Generator**
      - **Anthropic Model - Reply** → AI language model port

15. **Create the CRM/helpdesk HTTP node**
    - Add node type: **HTTP Request**.
    - Name it **Update CRM/Helpdesk**.
    - Set:
      - **Method:** `POST`
      - **URL:** `{{ $('Workflow Configuration').first().json.crmApiUrl }}`
    - Enable sending headers and body.
    - Add header:
      - `Content-Type: application/json`
    - Add body parameters:
      - `ticket_id` = `{{ $('Normalize Ticket Data').item.json.ticket_id }}`
      - `priority` = `{{ $('Support Intelligence Agent').item.json.urgency }}`
      - `tags` = `{{ $('Support Intelligence Agent').item.json.category }}`
      - `summary` = `{{ $('Support Intelligence Agent').item.json.summary }}`
      - `sentiment` = `{{ $('Support Intelligence Agent').item.json.sentiment }}`
      - `churn_risk` = `{{ $('Support Intelligence Agent').item.json.churn_risk }}`
      - `draft_reply` = `{{ $('Draft Reply Generator').item.json.output }}`
      - `original_language` = `{{ $('Language Detection & Translation Agent').item.json.original_language }}`
    - Connect **Draft Reply Generator** → **Update CRM/Helpdesk**.
    - If your CRM requires authentication, add the appropriate auth method here.

16. **Create the escalation webhook node**
    - Add node type: **HTTP Request**.
    - Name it **Escalate to Team**.
    - Set:
      - **Method:** `POST`
      - **URL:** `{{ $('Workflow Configuration').first().json.escalationWebhookUrl }}`
    - Enable sending headers and body.
    - Add header:
      - `Content-Type: application/json`
    - Add body parameters:
      - `ticket_id` = `{{ $json.ticket_id }}`
      - `urgency` = `{{ $json.urgency }}`
      - `category` = `{{ $json.category }}`
      - `sentiment` = `{{ $json.sentiment }}`
      - `summary` = `{{ $json.summary }}`
      - `churn_risk` = `{{ $json.churn_risk }}`
      - `user_email` = `{{ $json.user_email }}`
      - `original_message` = `{{ $json.original_message }}`
    - Connect **Decision Router** escalation output → **Escalate to Team**.

17. **Create the observability code node**
    - Add node type: **Code**.
    - Name it **Log Observability Metrics**.
    - Set mode to **Run Once for Each Item**.
    - Add logic to:
      - determine workflow start time
      - compute response time
      - identify whether the item was escalated
      - collect category, urgency, sentiment, churn risk
      - optionally POST to an observability endpoint
    - Connect:
      - **Update CRM/Helpdesk** → **Log Observability Metrics**
      - **Escalate to Team** → **Log Observability Metrics**

18. **Fix the configuration mismatches before production**
    - The provided workflow contains inconsistencies. Correct them while rebuilding:
      - Use the same property name everywhere for observability, preferably `observabilityEndpoint`
      - Use `Normalize Ticket Data.timestamp` instead of a nonexistent `Workflow Configuration.timestamp`
      - If you want correct escalation metrics, add a field such as `escalated = true` on the escalation branch or infer it in code from the incoming node name

19. **Optional but recommended hardening**
    - Add error handling or dedicated error branches for:
      - Anthropic API failures
      - invalid CRM responses
      - escalation webhook errors
    - Add input validation for webhook payloads
    - Consider a Merge or explicit Set node if downstream branches need a guaranteed unified schema

20. **Test both entry points**
    - Test with:
      - an English email
      - a non-English email
      - a webhook payload with minimal required fields
      - a critical issue that should escalate
      - a low-risk issue that should generate an auto-reply draft

### Expected input/output expectations

#### Expected inbound ticket shape
Useful fields the workflow expects from either source:
- `id` or `messageId`
- `from` or `email`
- `html`, `body`, or `text`
- optionally `source`

#### Translation agent expected output
- `original_language`
- `translated_text`
- `confidence`

#### Intelligence agent expected output
- `sentiment`
- `urgency`
- `category`
- `summary`
- `churn_risk`
- `recommended_path`

#### CRM/helpdesk request output expectation
Any successful HTTP response is accepted unless you add validation.

#### Escalation webhook output expectation
Any successful HTTP response is accepted unless you add validation.

#### Sub-workflow setup
- No sub-workflows are used in this workflow.
- No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow processes incoming support tickets from email or webhooks. Messages are cleaned, normalized, and translated to English if needed. AI agents analyze the ticket to determine sentiment, urgency, category, and churn risk. Based on the analysis, tickets are either auto-replied with a drafted response or escalated to a support team. Ticket data and AI insights are then sent to a CRM/helpdesk system. Finally, observability metrics are logged. | General workflow description |
| Setup steps from the canvas note: 1. Add Anthropic API credentials for the AI agents. 2. Configure IMAP credentials if using email ingestion. 3. Set your CRM/Helpdesk API URL in the Workflow Configuration node. 4. Add an escalation webhook URL for urgent tickets. 5. Optionally configure an observability endpoint for metrics logging. | General setup guidance |
| The model selected in all AI model nodes is Claude Sonnet 4.5 using identifier `claude-sonnet-4-5-20250929`. Availability depends on your Anthropic account and the n8n Anthropic integration version. | Anthropic integration note |
| The current implementation routes `engineering` recommendations into the fallback escalation path because no dedicated engineering branch exists. | Routing design caveat |
| The current observability implementation has naming bugs: `observabilityEndpoint` vs `observability_endpoint`, and start timestamp is read from the wrong node. | Important implementation caveat |