Generate AI sales proposals using Anthropic Claude, PDF Noodle and your CRM

https://n8nworkflows.xyz/workflows/generate-ai-sales-proposals-using-anthropic-claude--pdf-noodle-and-your-crm-13232


# Generate AI sales proposals using Anthropic Claude, PDF Noodle and your CRM

## 1. Workflow Overview

**Purpose:** This workflow generates a customized sales proposal PDF from CRM-provided customer/meeting data using an AI agent (Anthropic Claude), renders the proposal via **PDF Noodle (Pdforge node)**, downloads the resulting PDF, and emails it to the prospect via **Gmail**.

**Target use cases:**
- Automatically turning discovery call notes/transcripts into a structured proposal (summary, challenges, solutions, pricing, next steps).
- Sending a polished proposal minutes after a meeting, directly from CRM automation.

### 1.1 Input Reception & Field Normalization (CRM → n8n)
Receives a POST payload from a CRM (or any system), then maps/normalizes key fields (name/email/company) and packages all received data for AI processing.

### 1.2 AI Structuring (Transcript → Proposal JSON)
Uses an **AI Agent** powered by **Anthropic Claude** plus a **Structured Output Parser** to turn raw transcript + customer data into a JSON object that matches the PDF template’s expected variables.

### 1.3 PDF Generation & Retrieval (JSON → PDF binary)
Sends the structured JSON to **PDF Noodle / Pdforge** to generate a proposal PDF, then downloads the PDF as binary.

### 1.4 Email Delivery (PDF binary → Gmail)
Emails the prospect with a personalized message and attaches the generated PDF.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Field Normalization
**Overview:** Accepts an incoming webhook request and standardizes the fields used throughout the workflow (contact name, email, company), while preserving the full payload for AI context.

**Nodes Involved:**
- Webhook
- Fields Mapping

#### Node: **Webhook**
- **Type / role:** `n8n-nodes-base.webhook` — entry point trigger (HTTP endpoint).
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `/ai-proposal`
- **Key variables/expressions:** None (receives raw JSON body).
- **Connections:**
  - **Output →** Fields Mapping
- **Version-specific:** TypeVersion **2.1**.
- **Edge cases / failures:**
  - Missing expected fields (e.g., `contactEmail`) will later break expressions in Gmail node.
  - If your CRM sends non-JSON or different field names, the Set node mappings must be updated.
  - Webhook authentication is not configured here; endpoint may be publicly callable unless n8n instance-level protection is used.

#### Node: **Fields Mapping**
- **Type / role:** `n8n-nodes-base.set` — normalizes incoming fields and creates a single text blob for AI.
- **Configuration choices (assignments):**
  - `name` = `{{$json.contactName}}`
  - `email` = `{{$json.contactEmail}}`
  - `company` = `{{$json.clientCompanyName}}`
  - `available_data` = `{{ JSON.stringify($json) }}`
- **Key expressions/variables:**
  - `available_data` is the *entire inbound payload* serialized into a string for the AI prompt.
- **Connections:**
  - **Input ←** Webhook
  - **Output →** AI Agent
- **Version-specific:** TypeVersion **3.4**.
- **Edge cases / failures:**
  - If `contactName/contactEmail/clientCompanyName` don’t exist, later nodes that reference `$('Fields Mapping').first().json...` will fail or produce blank fields.
  - Very large transcripts may exceed model or node limits; consider truncation or file-based handling if needed.

---

### Block 2 — AI Structuring (Claude + structured parsing)
**Overview:** Feeds customer context and transcript to a LangChain-based AI Agent configured with a sales-proposal system message. A structured output parser is attached to force JSON output usable by the PDF generator.

**Nodes Involved:**
- AI Agent
- Anthropic Chat Model
- Structured Output Parser

#### Node: **Anthropic Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — provides the LLM backend to the agent.
- **Configuration choices:**
  - Model: **`claude-sonnet-4-5-20250929`** (as selected from list)
  - Uses **Anthropic API** credentials.
- **Connections:**
  - **Output (ai_languageModel) →** AI Agent (as its language model input)
- **Version-specific:** TypeVersion **1.3**.
- **Edge cases / failures:**
  - Auth/credential issues (invalid API key, revoked access).
  - Model availability changes (the configured model name must exist in your Anthropic account/region).
  - Rate limits / timeouts on large transcripts.

#### Node: **Structured Output Parser**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces/validates JSON structure from the agent.
- **Configuration choices:**
  - `jsonSchemaExample`: currently set to a placeholder: **`{ ... insert your variable schema here }`**
- **Connections:**
  - **Output (ai_outputParser) →** AI Agent (as its output parser)
- **Version-specific:** TypeVersion **1.3**.
- **Edge cases / failures:**
  - As-is, the schema is not a real schema/example; the agent may output inconsistent JSON, or the parser may not validate as expected.
  - If the parser is strict, malformed JSON from the model will cause runtime failure.
- **Important integration note:** This node must reflect the **exact variable schema** required by your PDF Noodle template (field names and nesting).

#### Node: **AI Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model + parsing to produce structured proposal data.
- **Configuration choices:**
  - **Prompt text:**  
    `Here's the information about the customer I want you to use to generate the expected output:\n{{ $json.available_data }}`
  - **System message:** Sales proposal assistant instructions:
    - Extract: Challenges (max 3), Solutions (max 3), Pricing, Executive summary, Next steps
    - Return **ONLY valid JSON** matching schema
    - Make reasonable assumptions if missing
    - Price formatting rules
  - `promptType`: define
  - `hasOutputParser`: true (wired to Structured Output Parser)
- **Key expressions/variables:**
  - Uses `{{$json.available_data}}` from Fields Mapping.
- **Connections:**
  - **Input ←** Fields Mapping
  - **AI language model input ←** Anthropic Chat Model
  - **AI output parser input ←** Structured Output Parser
  - **Main output →** Generate PDF syncronously
- **Version-specific:** TypeVersion **3.1**.
- **Edge cases / failures:**
  - If transcript is missing or too vague, the “reasonable assumptions” rule may create inaccurate proposals; consider adding guardrails (e.g., require certain fields).
  - If the schema is mismatched with the PDF template variables, PDF generation may fail downstream.
  - Model may occasionally return non-JSON unless schema/parser is correctly configured and strict.

---

### Block 3 — PDF Generation & Download
**Overview:** Converts the AI-produced structured JSON into a PDF using PDF Noodle (Pdforge), then downloads the PDF from a signed URL into binary format for emailing.

**Nodes Involved:**
- Generate PDF syncronously
- Download PDF binary

#### Node: **Generate PDF syncronously**
- **Type / role:** `n8n-nodes-pdforge.pdforge` — generates a PDF from a template using provided variables (PDF Noodle / Pdforge integration).
- **Configuration choices:**
  - Template ID: `7f22f51f23`
  - Variables: `{{ JSON.stringify($json.output) }}`
    - Assumes the AI Agent’s result includes an `output` object containing the template variables.
- **Connections:**
  - **Input ←** AI Agent
  - **Output →** Download PDF binary
- **Version-specific:** TypeVersion **1**.
- **Edge cases / failures:**
  - If AI output does not include `$json.output`, variables become `undefined` → PDF generation likely fails.
  - Template variable names must match exactly; missing required template variables may error or produce blank fields.
  - Pdforge credentials or template access issues.
  - “Synchronously” implies waiting for render; could time out on large/complex templates.

#### Node: **Download PDF binary**
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches the generated PDF file.
- **Configuration choices:**
  - URL: `{{$json.signedUrl}}` (expects Pdforge node output to include `signedUrl`)
  - Default options (no explicit “Download as file” setting shown; behavior depends on node defaults/version).
- **Connections:**
  - **Input ←** Generate PDF syncronously
  - **Output →** Send a message
- **Version-specific:** TypeVersion **4.3**.
- **Edge cases / failures:**
  - If `signedUrl` is missing/expired, request fails (403/404).
  - If not configured to store response as binary in the expected property, Gmail attachment may be empty/missing.
  - Network issues / TLS problems.

---

### Block 4 — Email Delivery (Gmail + attachment)
**Overview:** Sends a personalized email to the prospect using Gmail and attaches the generated proposal PDF.

**Nodes Involved:**
- Send a message

#### Node: **Send a message**
- **Type / role:** `n8n-nodes-base.gmail` — sends outbound email via Gmail OAuth2.
- **Configuration choices:**
  - **To:** `{{ $('Fields Mapping').first().json.email }}`
  - **Subject:** `[Your Company] Proposal for {{ $('Fields Mapping').first().json.company }}`
  - **Body (text):** Personalized greeting using first name:  
    `{{ $('Fields Mapping').first().json.name.split(" ")[0] }}`
  - **Attachments:** uses UI-based binary attachments (`attachmentsBinary`), but no explicit binary property name is shown.
- **Connections:**
  - **Input ←** Download PDF binary
  - (No downstream nodes)
- **Version-specific:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - If `name` is empty, `split(" ")[0]` can throw or produce unexpected output.
  - If the PDF binary is not in the binary property expected by the Gmail node attachment settings, the email sends without attachment or fails.
  - Gmail OAuth token expiry/revocation; missing `gmail.send` scopes.
  - Sending limits/quota issues for the Gmail account.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook Trigger | Receives CRM POST payload | — | Fields Mapping | ## Welcome to Generate Custom Proposal with AI Workflow!  / **This workflow has the following sequence:** / 1. Webhook trigger (connected to your CRM) / 2. Map all the fields we're going to need on the workflow / 3. Transforming customer data and transcription into a formatted output / 4. Transform it into a custom PDF proposal using pdf noodle / 5. Sending the proposal through e-mail (You can change this nome to send using other formats as well) / **The following accesses are required for the workflow:** / - pdf noodle account: [Create here](https://app.pdfnoodle.com/auth/sign-up) / - AI API access (e.g. via OpenAI, Anthropic, Google or Ollama) / - Google OAuth2 Connected (with gmail permission): [Documentation](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/) / You can contact me via LinkedIn, if you have any questions: https://www.linkedin.com/in/marceloamiranda |
| Fields Mapping | Set | Maps CRM fields to canonical names + serializes payload for AI | Webhook | AI Agent | ### 1. Receiving and mapping data from CRM |
| AI Agent | LangChain Agent | Extracts proposal structure from transcript/data as JSON | Fields Mapping; Anthropic Chat Model (ai_languageModel); Structured Output Parser (ai_outputParser) | Generate PDF syncronously | ### 2. AI Agents formatting the data into the expected format for the PDF generation |
| Anthropic Chat Model | LangChain Anthropic Chat Model | Provides Claude model to the agent | — | AI Agent | ### 2. AI Agents formatting the data into the expected format for the PDF generation |
| Structured Output Parser | LangChain Structured Output Parser | Enforces schema/JSON structure for agent output | — | AI Agent | ### 2. AI Agents formatting the data into the expected format for the PDF generation |
| Generate PDF syncronously | Pdforge (PDF Noodle) | Renders PDF from template + variables | AI Agent | Download PDF binary | ### 3. Generating and downloading the PDF of the proposal |
| Download PDF binary | HTTP Request | Downloads PDF via signed URL to binary | Generate PDF syncronously | Send a message | ### 3. Generating and downloading the PDF of the proposal |
| Send a message | Gmail | Emails prospect and attaches generated PDF | Download PDF binary | — | ### 4. Sending an e-mail to the prospect with the PDF Proposal attached to it |
| Sticky Note | Sticky Note | Documentation / onboarding note | — | — | ## Welcome to Generate Custom Proposal with AI Workflow! / (content includes links above) |
| Sticky Note1 | Sticky Note | Block label | — | — | ### 1. Receiving and mapping data from CRM |
| Sticky Note2 | Sticky Note | Video reference | — | — | @[youtube](yRlHu5yvuSA) |
| Sticky Note3 | Sticky Note | Block label | — | — | ### 2. AI Agents formatting the data into the expected format for the PDF generation |
| Sticky Note4 | Sticky Note | Block label | — | — | ### 3. Generating and downloading the PDF of the proposal |
| Sticky Note5 | Sticky Note | Block label | — | — | ### 4. Sending an e-mail to the prospect with the PDF Proposal attached to it |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Webhook trigger**
   1. Add node: **Webhook**
   2. Set **HTTP Method** = `POST`
   3. Set **Path** = `ai-proposal`
   4. Save the workflow to obtain a production URL.
   5. (Optional but recommended) Add webhook authentication or protect the endpoint at n8n/proxy level.

2) **Add field normalization**
   1. Add node: **Set** (rename to **Fields Mapping**)
   2. Add fields:
      - `name` (String) = expression `{{$json.contactName}}`
      - `email` (String) = expression `{{$json.contactEmail}}`
      - `company` (String) = expression `{{$json.clientCompanyName}}`
      - `available_data` (String) = expression `{{ JSON.stringify($json) }}`
   3. Connect: **Webhook → Fields Mapping**

3) **Create the AI components (Claude + parser + agent)**
   1. Add node: **Anthropic Chat Model**
      - Choose model similar to: `claude-sonnet-4-5-20250929` (or an available Sonnet model in your account)
      - Create/select **Anthropic API credentials**
   2. Add node: **Structured Output Parser**
      - Replace the placeholder with a **real JSON schema example** matching your PDF template variables.
      - Example concept (you must adapt to your template): a top-level object containing `executive_summary`, arrays for `challenges`, `solutions`, `pricing`, `next_steps`, etc.
   3. Add node: **AI Agent**
      - In prompt text, set expression:  
        `Here's the information about the customer I want you to use to generate the expected output:\n{{ $json.available_data }}`
      - Add the **System Message** instructing extraction and “return only valid JSON”.
      - Ensure “Use Output Parser” (or equivalent) is enabled.
   4. Wire the AI inputs:
      - Connect **Anthropic Chat Model** to **AI Agent** via the **ai_languageModel** connection.
      - Connect **Structured Output Parser** to **AI Agent** via the **ai_outputParser** connection.
   5. Connect main flow: **Fields Mapping → AI Agent**

4) **Set up PDF Noodle / Pdforge generation**
   1. Add node: **Pdforge** (rename to **Generate PDF syncronously**)
   2. Configure **Credentials**: connect your **PDF Noodle/Pdforge account**.
   3. Set **Template ID** to your template (e.g., `7f22f51f23`).
   4. Set **Variables** to expression:  
      `{{ JSON.stringify($json.output) }}`
      - Important: ensure the AI Agent output actually contains `output` with the correct structure. If your agent returns the object at the root, adjust to `JSON.stringify($json)` or map it accordingly.
   5. Connect: **AI Agent → Generate PDF syncronously**

5) **Download the generated PDF**
   1. Add node: **HTTP Request** (rename to **Download PDF binary**)
   2. Set **URL** to expression: `{{$json.signedUrl}}`
   3. Configure the node to **download the response as a file/binary** and store it in a known binary property (commonly `data`), depending on your n8n HTTP Request node UI.
   4. Connect: **Generate PDF syncronously → Download PDF binary**

6) **Send the email with Gmail**
   1. Add node: **Gmail** (rename to **Send a message**)
   2. Configure **Gmail OAuth2 credentials** (Google project + OAuth consent + scopes allowing send).
   3. Set **To**: `{{ $('Fields Mapping').first().json.email }}`
   4. Set **Subject**: `[Your Company] Proposal for {{ $('Fields Mapping').first().json.company }}`
   5. Set **Message** body and include greeting:  
      `Hey {{ $('Fields Mapping').first().json.name.split(" ")[0] }}, ...`
   6. Attach the PDF:
      - In **Attachments**, select the binary property produced by **Download PDF binary** (e.g., `data`).
      - Ensure the attachment field points to the correct binary name; otherwise, the email will not include the PDF.
   7. Connect: **Download PDF binary → Send a message**

7) **Test end-to-end**
   1. Execute the workflow with a sample payload (like the pinned Webhook data).
   2. Verify:
      - AI output is valid JSON and matches the template variables.
      - Pdforge returns a `signedUrl`.
      - HTTP Request stores binary correctly.
      - Gmail node includes the attachment and sends successfully.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PDF Noodle account creation | https://app.pdfnoodle.com/auth/sign-up |
| Google OAuth2 (generic) documentation | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/ |
| Contact (LinkedIn) | https://www.linkedin.com/in/marceloamiranda |
| Video reference embedded in sticky note | @[youtube](yRlHu5yvuSA) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.