Qualify Idealista leads via WhatsApp with Superchat and GPT-4o

https://n8nworkflows.xyz/workflows/qualify-idealista-leads-via-whatsapp-with-superchat-and-gpt-4o-13524


# Qualify Idealista leads via WhatsApp with Superchat and GPT-4o

## 1. Workflow Overview

**Purpose:** Automatically detect incoming **Idealista** lead emails (Spain/Italy/Portugal real-estate platform), extract structured lead data using **GPT-4o**, then immediately send a **WhatsApp template message** via **Superchat** to start qualification.

**Target use cases:**
- Real-estate agencies/agents receiving Idealista enquiries by email who want instant WhatsApp outreach
- Teams wanting standardized lead capture (name/phone/property link/ref) and fast response time

### 1.1 Inbox Monitoring (3 alternative entry points)
Monitors incoming emails via **Gmail**, **Microsoft Outlook**, or **IMAP**. You typically keep **one** trigger and remove the others.

### 1.2 Idealista Detection (Email Filtering)
Filters messages to keep only those that look like they come from Idealista.

### 1.3 AI Extraction (LLM + Structured JSON)
Passes the raw email payload to an AI Agent configured to output **strict JSON** matching a given schema, using:
- **OpenAI Chat Model (gpt-4o)**
- **Structured Output Parser** (schema enforcement)

### 1.4 WhatsApp Outreach (Superchat)
Uses the extracted `phone` field to send an approved **WhatsApp Template** message through Superchat.

---

## 2. Block-by-Block Analysis

### Block 1 — Inbox Monitoring (Entry Points)

**Overview:** Provides three alternative ways to receive new inbound lead emails. All three feed into the same filter node.

**Nodes involved:**
- Gmail Trigger
- Microsoft Outlook Trigger
- Email Trigger (IMAP)
- Sticky Note / Sticky Note4 / Sticky Note9 (documentation-only)

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — polls Gmail for new emails.
- **Key configuration:**
  - Polling: **every minute**
  - Filters: none (captures broadly, filtering happens later)
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Outputs:** Main output → **from Idealista?**
- **Edge cases / failures:**
  - OAuth token expiry / missing scopes
  - Gmail API quota/rate limits
  - Polling duplicates if Gmail labels/history are not stable (depends on Gmail trigger internals)

#### Node: Microsoft Outlook Trigger
- **Type / role:** `n8n-nodes-base.microsoftOutlookTrigger` — polls Outlook mailbox.
- **Key configuration:**
  - Polling: **every hour**
  - Filters/options: default/empty
- **Credentials:** Microsoft Outlook OAuth2
- **Outputs:** Main output → **from Idealista?**
- **Edge cases / failures:**
  - OAuth consent/scopes issues (Graph permissions)
  - Slower polling (up to 1 hour delay)
  - Tenant admin restrictions can block mailbox access

#### Node: Email Trigger (IMAP)
- **Type / role:** `n8n-nodes-base.emailReadImap` — reads email via IMAP (generic provider support).
- **Key configuration:** Default options (no explicit mailbox/search constraints shown)
- **Credentials:** Not included in JSON; must be configured (IMAP host/user/password or OAuth depending on provider)
- **Outputs:** Main output → **from Idealista?**
- **Edge cases / failures:**
  - IMAP connection/auth failures
  - TLS/port misconfiguration
  - Re-reading the same messages if “mark as read” / UID tracking is not configured as expected

#### Sticky Notes in this block (documentation)
- **Sticky Note (large overview):** Explains the end-to-end goal and what fields are extracted and why.
- **Sticky Note4:** “STEP 1 Connect your email inbox … Delete/Remove the others”
- **Sticky Note9:** “STEP 1” label for the area

---

### Block 2 — Idealista Detection (Filter)

**Overview:** Checks if the incoming email appears related to Idealista by searching for `idealista.com` in key header fields.

**Nodes involved:**
- from Idealista?

#### Node: from Idealista?
- **Type / role:** `n8n-nodes-base.filter` — gates the workflow to Idealista messages only.
- **Key configuration (interpreted):**
  - Condition: `contains("idealista.com")`
  - Field checked (expression): `{{ $json.from || $json.From || $json.Subject }}`
    - It tries `from`, then `From`, then `Subject` depending on which exists in the trigger output.
- **Input connections:** From any of the email triggers (Gmail / Outlook / IMAP).
- **Output connections:** If condition passes → **AI Agent extracts Lead Information**
- **Edge cases / failures:**
  - False negatives if Idealista emails do not include `idealista.com` in those fields (e.g., localized sender domains, different From formatting)
  - False positives if unrelated emails contain `idealista.com` in subject/body but not actually leads
  - Expression strictness: if none of `from`, `From`, `Subject` exists, left side becomes `undefined` and the contains check may fail (usually results in “does not match”, but depends on node behavior)

---

### Block 3 — AI Extraction (GPT-4o + Structured Output)

**Overview:** Sends the raw email JSON to an AI Agent instructed to extract lead fields into a strict JSON structure. Uses an output parser to enforce schema.

**Nodes involved:**
- AI Agent extracts Lead Information
- OpenAI Chat Model
- Structured Output Parser
- Sticky Note5 / Sticky Note8 (documentation-only)

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider node supplying the chat model to the agent.
- **Key configuration:**
  - Model: **gpt-4o**
  - Built-in tools: none enabled
- **Credentials:** OpenAI API key credential (“Superchat Rate Limited Key Growth Team”)
- **Connections:**
  - Provides `ai_languageModel` input to **AI Agent extracts Lead Information**
- **Edge cases / failures:**
  - OpenAI auth failure / invalid key
  - Rate limits / 429 errors (credential name suggests rate-limited key)
  - Model availability changes

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — constrains/validates LLM output to match a JSON schema example.
- **Key configuration:**
  - Schema example includes fields: `firstName`, `lastName`, `phone`, `email`, `propertyRef`, `propertyTitle`, `propertyAddress`, `propertyLink`, `message`, `timestamp`, `source`
- **Connections:**
  - Provides `ai_outputParser` input to **AI Agent extracts Lead Information**
- **Edge cases / failures:**
  - Parser failure if the model outputs invalid JSON or deviates from schema
  - Non-idealista emails: agent is instructed to return `{}`; downstream nodes may then fail if required fields (like phone) are missing

#### Node: AI Agent extracts Lead Information
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model + output parser to produce structured lead JSON.
- **Key configuration:**
  - **Prompt input (`text`):**  
    `Here is the email:\n{{JSON.stringify($json)}}`  
    (passes the entire email object, not just body text)
  - **System message:** extractor instructions including:
    - Output **only valid JSON** (no code blocks)
    - Use specified keys; missing values must be empty string (not null/unknown)
    - Normalize phone to include **+34** if missing
    - Detect name phrasing in Spanish (Hola, soy… / Me llamo… / Mi nombre es…)
    - Extract Idealista listing link pattern `idealista.com/inmueble/...`
    - If not Idealista lead: output `{}`  
  - `hasOutputParser`: true (uses Structured Output Parser)
- **Input connections:**
  - Main input from **from Idealista?**
  - `ai_languageModel` from **OpenAI Chat Model**
  - `ai_outputParser` from **Structured Output Parser**
- **Output connections:** Main output → **Send WhatsApp Template**
- **Edge cases / failures:**
  - If the incoming email JSON is large (attachments, long HTML), token usage may spike or cause truncation
  - Phone normalization assumes Spain (+34); this may be wrong for Portugal/Italy leads
  - If agent outputs `{}`, downstream WhatsApp send likely fails because `phone` is undefined/empty
  - Variations in email field naming between triggers (Gmail vs Outlook vs IMAP) may reduce extraction accuracy if body text isn’t present where the model expects it

#### Sticky Notes in this block (documentation)
- **Sticky Note5:** “STEP 2 Connect your OpenAI API Key or connect a different LLM”
- **Sticky Note8:** “STEP 2” label for the AI area

---

### Block 4 — WhatsApp Template Outreach (Superchat)

**Overview:** Sends a WhatsApp template message to the extracted phone number using Superchat.

**Nodes involved:**
- Send WhatsApp Template
- Sticky Note6 / Sticky Note7 / Sticky Note10 / Sticky Note11 (documentation-only)

#### Node: Send WhatsApp Template
- **Type / role:** `n8n-nodes-superchat.superchat` — sends a WhatsApp template message via Superchat API.
- **Key configuration:**
  - Resource: **message**
  - Operation: **sendWhatsAppTemplate**
  - Channel: a selected WhatsApp channel (example cached name “WhatsApp: +1234567890”)
  - Template: selected template (example cached name “2_Kreditanfrage”)
  - **Identifier (recipient):** `{{ $json.phone }}`
  - Template variables: currently empty (`variables.values: []`) — must be filled if the template requires placeholders
- **Credentials:** Superchat API key (“Superchat Demo22”)
- **Inputs:** Main input from **AI Agent extracts Lead Information**
- **Outputs:** None downstream
- **Edge cases / failures:**
  - Invalid/empty phone number → send fails
  - Template not approved or wrong language/namespace → send fails
  - Missing required template variables → API error
  - Channel/template IDs are workspace-specific; importing workflow requires re-selecting them

#### Sticky Notes in this block (documentation)
- **Sticky Note6:** “STEP 3 Connect your Superchat Workspace through an API Key”  
  Link: https://help.superchat.com/en/articles/213219-introduction-to-integrations#h_56604e4ec3
- **Sticky Note7:** “STEP 4 Create a WhatsApp Template…”  
  Link: https://help.superchat.com/en/articles/25035-get-started-whatsapp-templates
- **Sticky Note10:** “STEP 3 & 4” label for the Superchat area
- **Sticky Note11:** Example template text and example buttons

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Entry point: poll Gmail for new emails | — | from Idealista? | ## STEP 1  \nConnect your email inbox\n- Gmail OR\n- Outlook OR\n- IMAP (All email providers)\n\nDelete/Remove the others |
| Microsoft Outlook Trigger | n8n-nodes-base.microsoftOutlookTrigger | Entry point: poll Outlook for new emails | — | from Idealista? | ## STEP 1  \nConnect your email inbox\n- Gmail OR\n- Outlook OR\n- IMAP (All email providers)\n\nDelete/Remove the others |
| Email Trigger (IMAP) | n8n-nodes-base.emailReadImap | Entry point: read emails via IMAP | — | from Idealista? | ## STEP 1  \nConnect your email inbox\n- Gmail OR\n- Outlook OR\n- IMAP (All email providers)\n\nDelete/Remove the others |
| from Idealista? | n8n-nodes-base.filter | Filter: keep only Idealista-related emails | Gmail Trigger; Microsoft Outlook Trigger; Email Trigger (IMAP) | AI Agent extracts Lead Information |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider (gpt-4o) for extraction | — | AI Agent extracts Lead Information (ai_languageModel) | ## STEP 2\nConnect your OpenAI API Key or connect a different LLM |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/parse structured JSON output | — | AI Agent extracts Lead Information (ai_outputParser) | ## STEP 2\nConnect your OpenAI API Key or connect a different LLM |
| AI Agent extracts Lead Information | @n8n/n8n-nodes-langchain.agent | Extract lead fields from email into JSON | from Idealista?; OpenAI Chat Model; Structured Output Parser | Send WhatsApp Template | ## How to Automatically Qualify Idealista Leads via WhatsApp Using an AI Agent  \n\nThis workflow automates the process of capturing incoming **property leads from Idealista** one of the leading real estate platforms in **Spain, Italy, and Portugal** and qualifying them instantly via **WhatsApp** with Superchat.\n\nThe workflow monitors your inbox for new emails and applies a filter to identify messages coming from **Idealista**. Once a matching email is detected, it is forwarded to an **AI agent** that parses the unstructured email content and extracts key lead information, including:\n\n* **Name** – the lead's full name\n* **Phone number** – the lead's contact number\n* **Email** – the lead's email address\n* **Property of interest** – the listing the lead is inquiring about\n\nOnce the lead data is structured, the workflow automatically sends a **WhatsApp template message** to the lead to initiate the qualification process ensuring the fastest response times and a professional first touchpoint.\n\nBy automating this flow, you eliminate manual email screening, reduce response times from hours to seconds, and ensure no Idealista lead slips through the cracks. |
| Send WhatsApp Template | n8n-nodes-superchat.superchat | Send WhatsApp template to lead via Superchat | AI Agent extracts Lead Information | — | ## STEP 3\nConnect your Superchat Workspace [through an API Key.](https://help.superchat.com/en/articles/213219-introduction-to-integrations#h_56604e4ec3) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | ## How to Automatically Qualify Idealista Leads via WhatsApp Using an AI Agent  \n\nThis workflow automates the process of capturing incoming **property leads from Idealista** one of the leading real estate platforms in **Spain, Italy, and Portugal** and qualifying them instantly via **WhatsApp** with Superchat.\n\nThe workflow monitors your inbox for new emails and applies a filter to identify messages coming from **Idealista**. Once a matching email is detected, it is forwarded to an **AI agent** that parses the unstructured email content and extracts key lead information, including:\n\n* **Name** – the lead's full name\n* **Phone number** – the lead's contact number\n* **Email** – the lead's email address\n* **Property of interest** – the listing the lead is inquiring about\n\nOnce the lead data is structured, the workflow automatically sends a **WhatsApp template message** to the lead to initiate the qualification process ensuring the fastest response times and a professional first touchpoint.\n\nBy automating this flow, you eliminate manual email screening, reduce response times from hours to seconds, and ensure no Idealista lead slips through the cracks. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 1\nConnect your email inbox\n- Gmail OR\n- Outlook OR\n- IMAP (All email providers)\n\nDelete/Remove the others |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 2\nConnect your OpenAI API Key or connect a different LLM |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 3\nConnect your Superchat Workspace [through an API Key.](https://help.superchat.com/en/articles/213219-introduction-to-integrations#h_56604e4ec3) |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 4\n[Create a WhatsApp Template](https://help.superchat.com/en/articles/25035-get-started-whatsapp-templates) to reach out to your Idealista Leads.\n\n1. Select your WhatsApp Channel in the Superchat Node\n2. Select your WhatsApp template (when it's approved)\n3. Fill out your variables in chronological order |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 2 |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 1 |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation | — | — | ## STEP 3 & 4 |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Example Template\n\nHola {{firstName}}, gracias por tu interés en la referencia {{propertyTitle}} en la direccion {{propertyAddress}}.\n\nAquí tienes el enlace del anuncio: {{propertyLink}}.\n\nPara poder ayudarte mejor por favor elige una opcion:\n\nButton: Saber los requisitos\nButton: Agendar una visita\nButton: Conocer mas pisos |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add ONE email trigger (choose one, delete the others):**
   - Option A: **Gmail Trigger**
     - Set **Poll Times**: every minute
     - Connect **Gmail OAuth2** credentials
   - Option B: **Microsoft Outlook Trigger**
     - Set **Poll Times**: every hour
     - Connect **Microsoft Outlook OAuth2** credentials
   - Option C: **Email Trigger (IMAP)** (`Email Read IMAP`)
     - Configure IMAP host/port/TLS, username/password (or provider-specific auth)
     - Ensure message handling avoids duplicate processing (mark as read / move folder, if your setup requires it)

3. **Add a Filter node** named **“from Idealista?”**
   - Condition: **String → contains**
   - Left value expression:  
     `{{ $json.from || $json.From || $json.Subject }}`
   - Right value: `idealista.com`
   - Connect the chosen email trigger → **from Idealista?**

4. **Add the OpenAI model node**
   - Node: **OpenAI Chat Model** (`lmChatOpenAi`)
   - Model: **gpt-4o**
   - Configure **OpenAI API credentials**
   - (No tools required)

5. **Add the Structured Output Parser**
   - Node: **Structured Output Parser** (`outputParserStructured`)
   - Provide a schema example like:
     - `firstName`, `lastName`, `phone`, `email`, `propertyRef`, `propertyTitle`, `propertyAddress`, `propertyLink`, `message`, `timestamp`, `source`

6. **Add the AI Agent node**
   - Node: **AI Agent** (`agent`)
   - Prompt type: “define” (custom prompt)
   - Text field:  
     `Here is the email:\n{{ JSON.stringify($json) }}`
   - System message: paste instructions to:
     - output only valid JSON (no extra text)
     - fill the schema keys
     - empty strings for missing fields
     - phone normalization (+34 if missing)
     - if not Idealista lead → return `{}`  
   - Enable “has output parser” / structured output usage.

7. **Wire the AI components**
   - Connect **from Idealista?** → **AI Agent** (main)
   - Connect **OpenAI Chat Model** → **AI Agent** (as **ai_languageModel**)
   - Connect **Structured Output Parser** → **AI Agent** (as **ai_outputParser**)

8. **Add Superchat node to send WhatsApp template**
   - Node: **Superchat**
   - Resource: **Message**
   - Operation: **Send WhatsApp Template**
   - Select your **WhatsApp channel** in Superchat
   - Select your **approved WhatsApp template**
   - Recipient/identifier expression:  
     `{{ $json.phone }}`
   - Fill **template variables** in the exact order required by the template (if your template includes placeholders like firstName/propertyLink).
   - Configure **Superchat API Key** credentials (from Superchat workspace).

9. **Connect AI Agent → Superchat**
   - **AI Agent (main output)** → **Send WhatsApp Template**

10. **Test end-to-end**
   - Send yourself a real Idealista lead email sample (or a sanitized copy) into the monitored inbox.
   - Verify:
     - Filter passes
     - Agent outputs JSON with non-empty `phone`
     - Superchat sends template successfully

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Connect Superchat workspace via API key | https://help.superchat.com/en/articles/213219-introduction-to-integrations#h_56604e4ec3 |
| Create/approve WhatsApp templates in Superchat | https://help.superchat.com/en/articles/25035-get-started-whatsapp-templates |
| Example WhatsApp template text and buttons are provided in the canvas | (Sticky Note “Example Template”) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.