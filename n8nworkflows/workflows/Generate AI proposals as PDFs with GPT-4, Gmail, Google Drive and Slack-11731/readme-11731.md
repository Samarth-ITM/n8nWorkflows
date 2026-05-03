Generate AI proposals as PDFs with GPT-4, Gmail, Google Drive and Slack

https://n8nworkflows.xyz/workflows/generate-ai-proposals-as-pdfs-with-gpt-4--gmail--google-drive-and-slack-11731


# Generate AI proposals as PDFs with GPT-4, Gmail, Google Drive and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI Proposal Generator  
**Stated title:** Generate AI proposals as PDFs with GPT-4, Gmail, Google Drive and Slack  
**Purpose:** Accept client project details via an HTTP webhook, compute pricing/timeline metadata, generate a tailored proposal using GPT‑4, format it into branded HTML, convert it to a PDF, then distribute it عبر:
- Email to the client (PDF attached)
- Google Drive upload (archive)
- Slack notification (team visibility)  
Finally, it returns a structured JSON response to the webhook caller after all distribution paths complete.

### 1.1 Input Reception & Metadata Calculation
Receives the client payload and derives standardized fields (prices, dates, proposal number).

### 1.2 AI Proposal Drafting (GPT‑4)
Uses OpenAI GPT‑4 with a proposal-writer system prompt and client context to produce a multi-section proposal.

### 1.3 Content Normalization & HTML Rendering
Extracts the model output and converts markdown-like sections/bullets into a styled HTML document.

### 1.4 PDF Conversion & Multi-channel Distribution
Converts HTML → PDF, then:
- Emails PDF to the client via Gmail
- Uploads PDF to Google Drive
- Posts a Slack message with summary details

### 1.5 Completion & Webhook Response
Merges the three distribution branches and returns a success JSON payload.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Metadata Calculation
**Overview:** Accepts client project data via a POST webhook and computes pricing range, schedule dates, and a unique proposal identifier used throughout the workflow.  
**Nodes involved:** `Webhook`, `Define Pricing & Timeline Logic`

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — workflow entry point (HTTP endpoint).
- **Configuration (interpreted):**
  - **Method:** POST
  - **Path:** `proposal-generator` (endpoint becomes `/webhook/proposal-generator` or `/webhook-test/proposal-generator` depending on mode)
- **Key fields expected in request body** (used later via expressions):
  - `client_name`
  - `client_email`
  - `project_type`
  - `project_description`
  - `budget_range` (string like `"1000-3000"`)
  - `timeline_weeks` (number or numeric string)
- **Connections:**
  - Output → `Define Pricing & Timeline Logic`
- **Failure/edge cases:**
  - Missing/invalid body keys will break downstream expressions (e.g., `budget_range.split('-')`).
  - If `timeline_weeks` is not numeric, date math can fail or produce unexpected results.

#### Node: Define Pricing & Timeline Logic
- **Type / role:** `n8n-nodes-base.set` — derives normalized proposal metadata.
- **Configuration choices:**
  - Parses `budget_range` using `.split('-')`:
    - `base_price = body.budget_range.split('-')[0]`
    - `max_price = body.budget_range.split('-')[1]`
  - Timeline strings and dates:
    - `delivery_timeline = "<timeline_weeks> weeks"`
    - `start_date = now + 1 week`, formatted `MMMM dd, yyyy`
    - `end_date = now + (timeline_weeks + 1) weeks`, formatted `MMMM dd, yyyy`
  - Proposal number:
    - `PROP-<yyyyMMdd>-<random 0..999>`
- **Key expressions/variables:**
  - `$json.body...` (from Webhook output)
  - `$now.plus(...)` and `$now.toFormat(...)` (n8n date helpers)
  - `parseInt($json.body.timeline_weeks)`
  - `Math.floor(Math.random() * 1000)`
- **Connections:**
  - Input ← `Webhook`
  - Output → `Generate Proposal Content`
- **Failure/edge cases:**
  - `budget_range` not containing `-` yields `undefined` max price.
  - Extra whitespace in `budget_range` can leak into prices (e.g., `"1000 - 3000"`). Consider `.trim()`.
  - `timeline_weeks` missing/NaN makes `parseInt` return `NaN`, breaking `$now.plus(...)`.

---

### Block 1.2 — AI Content Generation
**Overview:** Sends a structured prompt to GPT‑4 combining client inputs and computed metadata, requesting a proposal with fixed section headers.  
**Nodes involved:** `Generate Proposal Content`, `Extract AI Content`

#### Node: Generate Proposal Content
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via n8n’s LangChain-based node.
- **Configuration choices:**
  - **Model:** `gpt-4`
  - **Temperature:** `0.7` (balanced creativity and consistency)
  - **Messages:**
    - **System:** defines tone, constraints, and required section headers:
      - `## Executive Summary`, `## Project Scope`, `## Methodology`, `## Timeline & Milestones`, `## Investment Breakdown`, `## Why Choose Us`
    - **User:** injects client fields from `Webhook` and pricing/timeline fields from the Set node.
- **Key expressions/variables:**
  - Client fields via `$('Webhook').item.json.body...`
  - Budget/timeline via `$json.base_price`, `$json.max_price`, `$json.delivery_timeline`, etc. (coming from `Define Pricing & Timeline Logic`)
- **Connections:**
  - Input ← `Define Pricing & Timeline Logic`
  - Output → `Extract AI Content`
- **Version-specific requirements:**
  - Requires the n8n LangChain OpenAI node package available in your n8n instance (commonly present in recent n8n versions).
- **Failure/edge cases:**
  - OpenAI auth errors (invalid API key, quota exceeded).
  - Model name availability (some accounts/regions may not have `gpt-4`; may require `gpt-4o` / `gpt-4.1` etc.).
  - Output schema changes: downstream extraction assumes a specific response shape.

#### Node: Extract AI Content
- **Type / role:** `n8n-nodes-base.set` — normalizes the model output and centralizes fields needed later.
- **Configuration choices:**
  - Extracts AI text from:  
    `ai_content = $json.output[0].content[0].text`
  - Copies client and metadata fields into the current item:
    - `client_name`, `client_email`, `project_type`
    - `proposal_number`, `base_price`, `max_price`, `delivery_timeline`, `start_date`, `end_date`
- **Notable configuration issue:**
  - One field is named **`=client_name`** (leading `=`) instead of `client_name`. This is likely a mistake and can confuse downstream usage. The Code node reads `client_name`, so the separate correct `client_name` field is still present via other assignments, but the `=client_name` field is extraneous.
- **Connections:**
  - Input ← `Generate Proposal Content`
  - Output → `Generates HTML`
- **Failure/edge cases:**
  - If OpenAI node returns a different structure, `output[0].content[0].text` may be undefined causing expression errors.
  - Trailing whitespace in `ai_content` (`... }} ` includes a space) is harmless but unnecessary.

---

### Block 1.3 — HTML Rendering
**Overview:** Converts markdown-like AI text into HTML and injects it into a branded proposal template.  
**Nodes involved:** `Generates HTML`

#### Node: Generates HTML
- **Type / role:** `n8n-nodes-base.code` — builds final HTML string.
- **Configuration choices:**
  - Reads fields from `$input.first().json`:
    - `client_name`, `project_type`, `proposal_number`, `base_price`, `max_price`, `delivery_timeline`, `start_date`, `end_date`, `client_email`, `ai_content`
  - Implements `convertToHTML(text)` that:
    - Converts `## Header` → `<h2>`
    - Converts bullet points `- item` → `<li>item</li>` and wraps consecutive items in `<ul>`
    - Converts `**bold**` → `<strong>`
    - Splits paragraphs by blank lines and wraps in `<p>` unless already `<h2>` or `<ul>`
  - Outputs JSON containing:
    - `html` (the full HTML document)
    - `client_name`, `client_email`, `proposal_number`
- **Connections:**
  - Input ← `Extract AI Content`
  - Output → `HTML to PDF`
- **Failure/edge cases:**
  - The regex for wrapping `<li>` blocks into `<ul>` is broad; malformed AI output could produce nested/incorrect list HTML.
  - If the AI deviates from expected `##` headers, formatting will degrade (still usable, but less structured).
  - Potential HTML injection: AI content is inserted into HTML after minimal transformations; if the model outputs raw HTML, it will pass through.

---

### Block 1.4 — PDF Creation & Distribution
**Overview:** Converts the HTML to a PDF binary, then sends it through Gmail, stores it in Google Drive, and alerts Slack.  
**Nodes involved:** `HTML to PDF`, `Send Email with PDF Attachment`, `Upload file`, `Send a message`

#### Node: HTML to PDF
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — external HTML-to-PDF rendering.
- **Configuration choices:**
  - `html_content` from `{{$json.html}}`
  - Output as `file` with filename `data`
- **Credentials:** HTML to PDF account (HTMLCSS to PDF API).
- **Connections:**
  - Input ← `Generates HTML`
  - Outputs → three parallel nodes:
    - `Send Email with PDF Attachment`
    - `Upload file`
    - `Send a message`
- **Failure/edge cases:**
  - API auth errors / insufficient credits.
  - Rendering failures due to complex CSS or unsupported features.
  - Output binary property name must match what downstream nodes expect (see email/drive attachment handling).

#### Node: Send Email with PDF Attachment
- **Type / role:** `n8n-nodes-base.gmail` — sends client email with PDF attachment.
- **Configuration choices:**
  - **To:** `$('Webhook').item.json.body.client_email`
  - **Subject:** `Your Project Proposal - <project_type>`
  - **HTML Body:** a branded email template that references:
    - Client name, project type (from Webhook)
    - Timeline and prices, proposal number (from `Extract AI Content`)
  - **Attachments:** configured via `attachmentsBinary`, but the workflow JSON shows an empty object `{}` in the attachments array.
- **Critical edge case (attachment mapping):**
  - Gmail node attachments require selecting the **binary property name** that holds the PDF. Here it is not explicitly set, so the email may be sent **without the PDF** unless n8n auto-selects a default binary key.
  - You typically must set attachment binary property (often `data` or similar depending on the PDF node output).
- **Credentials:** Gmail OAuth2.
- **Connections:**
  - Input ← `HTML to PDF`
  - Output → `Merge` (input index 0)
- **Failure/edge cases:**
  - Gmail OAuth scope missing (send permission).
  - Attachment size limits.
  - Invalid recipient email.
  - HTML rendering differences in email clients.

#### Node: Upload file
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads the generated PDF.
- **Configuration choices:**
  - **File name:** `Proposal_<ClientName>_<ProposalNumber>.pdf` with spaces replaced by underscores in client name.
  - **Drive:** “My Drive”
  - **Folder:** `root` (UI shows a cached folder name placeholder `YOUR_FOLDER_NAME`)
- **Credentials:** Google Drive OAuth2.
- **Connections:**
  - Input ← `HTML to PDF`
  - Output → `Merge` (input index 1)
- **Failure/edge cases:**
  - Incorrect folder selection / insufficient permissions.
  - If the node is not configured to read the correct binary property, upload will fail or upload empty content.
  - Google API rate limits.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.slack` — sends a channel notification.
- **Configuration choices:**
  - Posts a formatted message including proposal number, client, project, budget, timeline, dates, and status.
  - **Channel selection:** by `channelId` (placeholder `YOUR_SLACK_CHANNEL_ID`).
- **Credentials:** Slack API.
- **Connections:**
  - Input ← `HTML to PDF`
  - Output → `Merge` (input index 2)
- **Failure/edge cases:**
  - Missing Slack scopes (`chat:write`, channel access).
  - Posting to a private channel without bot membership.
  - Rate limiting.

---

### Block 1.5 — Completion & Webhook Response
**Overview:** Waits for all three parallel branches (email, drive, slack) to finish, then returns a single JSON response to the webhook caller.  
**Nodes involved:** `Merge`, `Respond to Webhook`

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — synchronization gate for parallel branches.
- **Configuration choices:**
  - `numberInputs: 3` (expects three incoming streams)
  - Used to ensure downstream response occurs only after all branches run.
- **Connections:**
  - Inputs:
    - Index 0 ← `Send Email with PDF Attachment`
    - Index 1 ← `Upload file`
    - Index 2 ← `Send a message`
  - Output → `Respond to Webhook`
- **Failure/edge cases:**
  - If any branch errors and stops execution, Merge never completes (webhook caller may time out).
  - If a branch produces zero items, merge behavior can be surprising depending on merge mode; here it’s functioning mainly as a join barrier.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns HTTP response.
- **Configuration choices:**
  - Responds with JSON containing:
    - `success`, `message`
    - `data`: proposal metadata
    - `status`: booleans indicating each step “true” (static flags; not actually computed from branch results)
    - `timestamp: $now.toISO()`
- **Connections:**
  - Input ← `Merge`
- **Failure/edge cases:**
  - Response claims success even if email/upload/slack partially failed (unless failures stop execution).
  - Webhook timeout risk if PDF conversion or OpenAI call is slow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry point: receives client payload | — | Define Pricing & Timeline Logic | ## Input & Configuration<br>Receives client project details via webhook and calculates pricing, timeline, and proposal metadata. |
| Define Pricing & Timeline Logic | n8n-nodes-base.set | Derive prices, dates, proposal number | Webhook | Generate Proposal Content | ## Input & Configuration<br>Receives client project details via webhook and calculates pricing, timeline, and proposal metadata. |
| Generate Proposal Content | @n8n/n8n-nodes-langchain.openAi | Generate proposal text with GPT‑4 | Define Pricing & Timeline Logic | Extract AI Content | ## AI Content Generation<br>Uses OpenAI GPT-4 to write a customized proposal, then formats it into a professional HTML document. |
| Extract AI Content | n8n-nodes-base.set | Extract AI text and normalize fields | Generate Proposal Content | Generates HTML | ## AI Content Generation<br>Uses OpenAI GPT-4 to write a customized proposal, then formats it into a professional HTML document. |
| Generates HTML | n8n-nodes-base.code | Convert proposal text to branded HTML | Extract AI Content | HTML to PDF | ## AI Content Generation<br>Uses OpenAI GPT-4 to write a customized proposal, then formats it into a professional HTML document. |
| HTML to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Convert HTML to PDF binary | Generates HTML | Send Email with PDF Attachment; Upload file; Send a message | ## PDF Creation & Distribution<br>Converts HTML to PDF and delivers via email, Google Drive storage, and Slack team notification. |
| Send Email with PDF Attachment | n8n-nodes-base.gmail | Email PDF to client | HTML to PDF | Merge | ## PDF Creation & Distribution<br>Converts HTML to PDF and delivers via email, Google Drive storage, and Slack team notification. |
| Upload file | n8n-nodes-base.googleDrive | Store PDF in Drive | HTML to PDF | Merge | ## PDF Creation & Distribution<br>Converts HTML to PDF and delivers via email, Google Drive storage, and Slack team notification. |
| Send a message | n8n-nodes-base.slack | Notify team in Slack | HTML to PDF | Merge | ## PDF Creation & Distribution<br>Converts HTML to PDF and delivers via email, Google Drive storage, and Slack team notification. |
| Merge | n8n-nodes-base.merge | Join/synchronize parallel branches | Send Email with PDF Attachment; Upload file; Send a message | Respond to Webhook | ## Completion<br>Merges all execution paths and returns a structured JSON success response to the caller. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return JSON response to caller | Merge | — | ## Completion<br>Merges all execution paths and returns a structured JSON success response to the caller. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ### How It Works<br>This workflow automates proposal generation for freelancers and agencies. When a client submits their project details through the webhook, the workflow processes their budget range and timeline, then uses OpenAI GPT-4 to generate a customized, professional proposal with sections covering executive summary, scope, methodology, timeline, and pricing. The AI-generated content is formatted into a branded HTML template, converted to PDF, and delivered via three channels: emailed directly to the client with the PDF attached, uploaded to Google Drive for your records, and posted to Slack to notify your team. The entire process completes in seconds, ensuring clients receive polished proposals instantly while your team stays informed.<br><br>### Setup Steps<br>1. Connect required credentials: OpenAI API, HTML-to-PDF service, Gmail, Google Drive, and Slack.<br>2. Update the email template with your company name, contact details, and branding.<br>3. Configure the Google Drive folder ID where proposals should be stored.<br>4. Set your Slack channel for proposal notifications.<br>5. Copy the webhook URL and integrate it into your contact form or CRM.<br>6. Test with sample client data to verify all integrations work correctly.<br><br>### Customization<br>- Modify the OpenAI system prompt to match your writing style and industry focus.<br>- Adjust the HTML template styling to match your brand colors and logo.<br>- Customize email subject lines and body content for your business tone. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## Input & Configuration<br>Receives client project details via webhook and calculates pricing, timeline, and proposal metadata. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## AI Content Generation<br>Uses OpenAI GPT-4 to write a customized proposal, then formats it into a professional HTML document. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## PDF Creation & Distribution<br>Converts HTML to PDF and delivers via email, Google Drive storage, and Slack team notification. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## Completion<br>Merges all execution paths and returns a structured JSON success response to the caller. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **AI Proposal Generator**
   - (Optional) Set execution order to default (`v1`).

2) **Add “Webhook” node**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: **proposal-generator**
   - Save and copy the **Test URL** and **Production URL** for later integration.

3) **Add “Define Pricing & Timeline Logic” node**
   - Node type: **Set**
   - Add fields (string unless noted):
     - `base_price` = `{{$json.body.budget_range.split('-')[0]}}`
     - `max_price` = `{{$json.body.budget_range.split('-')[1]}}`
     - `delivery_timeline` = `{{$json.body.timeline_weeks}} weeks`
     - `start_date` = `{{$now.plus(1, 'week').toFormat('MMMM dd, yyyy')}}`
     - `end_date` = `{{$now.plus(parseInt($json.body.timeline_weeks) + 1, 'weeks').toFormat('MMMM dd, yyyy')}}`
     - `proposal_number` = `PROP-{{$now.toFormat('yyyyMMdd')}}-{{Math.floor(Math.random() * 1000)}}`
   - Connect: **Webhook → Define Pricing & Timeline Logic**

4) **Add “Generate Proposal Content” node (OpenAI / LangChain)**
   - Node type: **OpenAI (LangChain)**
   - Credentials: configure **OpenAI API** credential (API key)
   - Model: **gpt-4**
   - Temperature: **0.7**
   - Messages:
     - System message: paste the workflow’s system prompt (proposal writer constraints + required `##` sections).
     - User message: paste the client/context prompt using expressions:
       - `client_name`, `project_type`, `project_description` from `$('Webhook').item.json.body...`
       - prices/dates from current `$json` (coming from Set node)
   - Connect: **Define Pricing & Timeline Logic → Generate Proposal Content**

5) **Add “Extract AI Content” node**
   - Node type: **Set**
   - Add fields:
     - `ai_content` = `{{$json.output[0].content[0].text}}`
     - `client_name` = `{{$('Webhook').item.json.body.client_name}}`
     - `client_email` = `{{$('Webhook').item.json.body.client_email}}`
     - `project_type` = `{{$('Webhook').item.json.body.project_type}}`
     - `proposal_number` = `{{$('Define Pricing & Timeline Logic').item.json.proposal_number}}`
     - `base_price` = `{{$('Define Pricing & Timeline Logic').item.json.base_price}}`
     - `max_price` = `{{$('Define Pricing & Timeline Logic').item.json.max_price}}`
     - `delivery_timeline` = `{{$('Define Pricing & Timeline Logic').item.json.delivery_timeline}}`
     - `start_date` = `{{$('Define Pricing & Timeline Logic').item.json.start_date}}`
     - `end_date` = `{{$('Define Pricing & Timeline Logic').item.json.end_date}}`
   - Important: **do not** create a field named `=client_name`; use `client_name`.
   - Connect: **Generate Proposal Content → Extract AI Content**

6) **Add “Generates HTML” node**
   - Node type: **Code**
   - Language: JavaScript
   - Paste the provided JS that:
     - reads the fields
     - converts markdown-like content into HTML
     - returns `{ html, client_name, client_email, proposal_number }`
   - Connect: **Extract AI Content → Generates HTML**

7) **Add “HTML to PDF” node**
   - Node type: **HTML to PDF (HTMLCSS to PDF)**
   - Credentials: configure your **HTMLCSS to PDF** API credential
   - Set:
     - HTML content = `{{$json.html}}`
     - Output format = **file**
     - Output filename = **data**
   - Connect: **Generates HTML → HTML to PDF**

8) **Add “Send Email with PDF Attachment” node**
   - Node type: **Gmail**
   - Credentials: configure **Gmail OAuth2** (scopes must allow sending email)
   - To: `{{$('Webhook').item.json.body.client_email}}`
   - Subject: `Your Project Proposal - {{$('Webhook').item.json.body.project_type}}`
   - Message (HTML): paste the email template; keep expressions referencing:
     - `$('Webhook')...client_name`, `project_type`
     - `$('Extract AI Content')...delivery_timeline`, `base_price`, `max_price`, `proposal_number`
   - Attachment:
     - Configure the attachment to use the **binary property created by the PDF node** (commonly the node outputs binary under a key like `data`). Ensure the Gmail node attachment points to that exact binary property.
   - Connect: **HTML to PDF → Send Email with PDF Attachment**

9) **Add “Upload file” node**
   - Node type: **Google Drive**
   - Credentials: configure **Google Drive OAuth2**
   - Operation: **Upload**
   - Binary property: select the PDF binary (same key as above, typically `data`)
   - File name:
     - `Proposal_{{$('Webhook').item.json.body.client_name.replace(/\s+/g, '_')}}_{{$('Extract AI Content').item.json.proposal_number}}.pdf`
   - Drive: **My Drive**
   - Folder: select your target folder (or root)
   - Connect: **HTML to PDF → Upload file**

10) **Add “Send a message” node**
   - Node type: **Slack**
   - Credentials: configure **Slack API** (bot token with `chat:write`, and channel access)
   - Resource/operation: **Send Message**
   - Channel: pick your channel
   - Text: paste the formatted message content with expressions.
   - Connect: **HTML to PDF → Send a message**

11) **Add “Merge” node**
   - Node type: **Merge**
   - Set **Number of Inputs = 3**
   - Connect branches:
     - **Send Email with PDF Attachment → Merge (Input 1 / index 0)**
     - **Upload file → Merge (Input 2 / index 1)**
     - **Send a message → Merge (Input 3 / index 2)**

12) **Add “Respond to Webhook” node**
   - Node type: **Respond to Webhook**
   - Respond with: **JSON**
   - Response body: paste the JSON template referencing `Extract AI Content` + `Webhook` fields and `$now.toISO()`
   - Connect: **Merge → Respond to Webhook**

13) **Test end-to-end**
   - Call the webhook test URL with JSON body containing required fields.
   - Verify:
     - GPT output is generated
     - PDF is produced
     - Email includes the PDF attachment
     - Drive upload creates a valid PDF
     - Slack message posts successfully
     - Webhook returns success JSON

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates proposal generation: webhook intake → GPT‑4 proposal → HTML template → PDF → email + Drive + Slack → JSON response. | From the canvas note “How It Works”. |
| Setup: connect credentials (OpenAI, HTML-to-PDF, Gmail, Drive, Slack), customize branding in email + HTML, set Drive folder + Slack channel, integrate webhook URL into form/CRM, test with sample data. | From the canvas note “Setup Steps”. |
| Customization ideas: adjust OpenAI system prompt, adjust HTML styling, customize email tone/subject. | From the canvas note “Customization”. |