Generate AI website legal and accessibility compliance reports with OpenAI, Gmail and Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-website-legal-and-accessibility-compliance-reports-with-openai--gmail-and-google-drive-11747


# Generate AI website legal and accessibility compliance reports with OpenAI, Gmail and Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Website Compliance Checker  
**Purpose:** Accept a website URL (plus requester email and company name) via a webhook, fetch and sanitize the site HTML, ask OpenAI to produce a structured legal/accessibility compliance assessment, generate a branded HTML report, convert it to PDF, email it to the requester via Gmail, and archive the PDF in Google Drive.

**Typical use cases**
- Automated website compliance screening for agencies/consultants
- Lead magnet compliance scan (user submits URL + email)
- Internal monitoring / periodic re-checks (with external scheduling added)

### 1.1 Input Reception & Website Scan
Receives POST data (url, email, company_name), downloads the HTML, strips scripts/styles, and truncates content for AI.

### 1.2 AI Compliance Analysis & Normalization
Sends cleaned HTML to OpenAI with a strict JSON-only response contract; parses/normalizes the result and applies a fallback structure on parse failures.

### 1.3 Report Generation & PDF Conversion
Builds a full HTML report (scores, badges, issue lists) and converts it into a PDF binary.

### 1.4 Delivery & Storage
Emails the PDF to the requester and saves the PDF into Google Drive.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Website Scan

**Overview:** This block exposes an HTTP endpoint, retrieves the target website HTML, and preprocesses it so it’s safe and compact enough for LLM analysis.

**Nodes involved:**
- Webhook
- Fetch Website HTML
- Extract & Clean HTML

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — Entry point; receives scan requests.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `/compliance-check`
  - **Response mode:** `lastNode` (the webhook response will be whatever the final node returns; in this workflow, that effectively waits until email/drive steps finish unless changed).
- **Expected input payload:** In `body`:
  - `url` (string; required)
  - `email` (string; required for delivery)
  - `company_name` (string; used in report/email/filename)
- **Connections:**
  - Output → **Fetch Website HTML**
- **Version notes:** TypeVersion `2.1` (current webhook node behavior depends on n8n version; responseMode “lastNode” can increase request duration).
- **Edge cases / failures:**
  - Missing fields (`body.url`, `body.email`, `body.company_name`) will later break expressions or lead to empty report/email destination.
  - With `responseMode=lastNode`, long downstream processing (HTTP fetch + LLM + PDF API) can cause webhook client timeouts.

#### Node: Fetch Website HTML
- **Type / role:** `n8n-nodes-base.httpRequest` — Downloads the website HTML.
- **Configuration (interpreted):**
  - **URL:** `{{ $json.body.url }}` (from webhook request body)
  - **Timeout:** 30s
  - **Response format:** text (HTML returned into the node output; referenced later as `json.data`)
  - **Redirects:** enabled (default redirect handling configured)
  - **Allow unauthorized certs:** false
- **Connections:**
  - Input ← **Webhook**
  - Output → **Extract & Clean HTML**
- **Version notes:** TypeVersion `4.3`
- **Edge cases / failures:**
  - Non-200 responses, blocked user-agents, WAF challenges (Cloudflare), or HTML delivered via JS-only rendering.
  - Large pages; although later truncated, the request still downloads full content.
  - If the request returns non-HTML (PDF, JSON, binary), cleaning/parsing assumptions may degrade analysis.

#### Node: Extract & Clean HTML
- **Type / role:** `n8n-nodes-base.code` — Sanitizes HTML for AI and normalizes fields.
- **Key logic / configuration:**
  - Reads HTML from: `$input.first().json.data`
  - Reads URL/email/company_name from the Webhook node via node reference:
    - `$('Webhook').first().json.body.url`
    - `$('Webhook').first().json.body.email`
    - `$('Webhook').first().json.body.company_name`
  - Cleans HTML by:
    - Removing `<script>...</script>` blocks (regex)
    - Removing `<style>...</style>` blocks (regex)
    - Collapsing whitespace
    - Truncating to first **50,000 characters**
  - Outputs:
    - `url`, `original_url` (same value)
    - `html_content` (cleaned, truncated)
    - `email`, `company_name`
- **Connections:**
  - Input ← **Fetch Website HTML**
  - Output → **Analyze Compliance**
- **Version notes:** TypeVersion `2`
- **Edge cases / failures:**
  - Regex-based stripping can fail on malformed HTML or edge cases (nested tags, unusual encodings).
  - If `Fetch Website HTML` returns no `data`, this node throws.
  - Truncation may remove relevant footer links (privacy/terms) if they appear late in the HTML.

---

### Block 1.2 — AI Compliance Analysis & Normalization

**Overview:** This block prompts OpenAI to audit the HTML and return strict JSON. It then parses the AI output, adds metadata, and provides a fallback on parsing errors.

**Nodes involved:**
- Analyze Compliance
- Parse Compliance Results

#### Node: Analyze Compliance
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call to analyze compliance and produce structured JSON.
- **Configuration (interpreted):**
  - **Model:** `chatgpt-4o-latest`
  - **Temperature:** 0.3 (more deterministic)
  - **Max tokens:** 2000 (caps response length)
  - **System message:** Defines auditor role and required checks:
    1) Privacy policy indicators (GDPR/CCPA)  
    2) Cookie consent banner/management  
    3) Terms of Service  
    4) Accessibility cues (WCAG: alt text, ARIA)  
    5) Contact information  
    6) SSL enabled (from URL)  
    7) DPO mention (GDPR)  
    And enforces **“Return ONLY a valid JSON object”** with a fixed schema.
  - **User message:** Injects:
    - `Website URL: {{ $json.url }}`
    - `HTML Content: {{ $json.html_content }}`
- **Credentials:** OpenAI API credential (placeholder in JSON).
- **Connections:**
  - Input ← **Extract & Clean HTML**
  - Output → **Parse Compliance Results**
- **Version notes:** TypeVersion `2.1` (LangChain-based OpenAI node behavior differs from legacy OpenAI nodes; response shape matters for parsing).
- **Edge cases / failures:**
  - Model may still return non-JSON or wrap JSON in markdown fences despite instruction.
  - Token limits: long HTML can cause the prompt to be truncated or reduce response quality.
  - Network/auth issues with OpenAI; rate limits.

#### Node: Parse Compliance Results
- **Type / role:** `n8n-nodes-base.code` — Extracts JSON from LLM output and enriches it with webhook metadata.
- **Key logic / configuration:**
  - Reads text from: `$input.first().json.output[0].content[0].text`
    - This is tightly coupled to the OpenAI node’s output structure.
  - Removes markdown code fences:
    - `.replace(/```json\n?/g, '')`
    - `.replace(/```\n?/g, '')`
  - `JSON.parse()` the cleaned string.
  - On failure, emits a **fallback object** with:
    - All checks set to defaults (scores 0, found false)
    - `critical_issues` includes parse error
    - `recommendations` includes “Please retry the scan”
    - `error: true` and `raw_response` preserved
  - Enriches with:
    - `website_url`, `company_name`, `email` (from `$('Webhook').first().json.body`)
    - `scan_date` (ISO string), `scan_timestamp` (ms epoch)
    - `has_error` boolean
- **Connections:**
  - Input ← **Analyze Compliance**
  - Output → **Generate HTML Report**
- **Version notes:** TypeVersion `2`
- **Edge cases / failures:**
  - If OpenAI output schema changes (e.g., different nesting), `chatGptOutput` extraction will throw.
  - If AI returns trailing commas or comments, JSON.parse fails (fallback triggers).
  - If webhook node data is unavailable (e.g., node renamed), node references break.

---

### Block 1.3 — Report Generation & PDF Conversion

**Overview:** Converts normalized compliance data into a styled HTML report, then calls an external service to convert HTML to a PDF binary.

**Nodes involved:**
- Generate HTML Report
- HTML to PDF

#### Node: Generate HTML Report
- **Type / role:** `n8n-nodes-base.code` — Produces a full branded HTML document.
- **Key logic / configuration:**
  - Builds helpers:
    - `getScoreColor(score)` returns gradient based on score thresholds (>=70 green, >=40 purple/red, else yellow/red)
    - `getScoreBadge(score)` maps to CSS classes `score-high|medium|low`
    - `formatDate(iso)` renders US locale date/time
  - Creates a complete HTML page including:
    - Overall score circle
    - Sections: Privacy Policy, Cookie Consent, Terms, Accessibility (with issues list), Contact Info, SSL
    - Conditional sections:
      - Critical Issues (if `critical_issues.length > 0`)
      - Recommendations (if `recommendations.length > 0`)
    - Footer disclaimer text
  - Output: original JSON plus `html_report` string.
- **Connections:**
  - Input ← **Parse Compliance Results**
  - Output → **HTML to PDF**
- **Version notes:** TypeVersion `2`
- **Edge cases / failures:**
  - If any expected fields are missing (e.g., `privacy_policy.score`), template string access can throw.
  - Very long `details` fields can inflate HTML and affect PDF conversion.

#### Node: HTML to PDF
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — External HTML-to-PDF conversion (HTMLCSS to PDF service).
- **Configuration (interpreted):**
  - **HTML input:** `{{ $json.html_report }}`
  - **Output format:** file (binary output)
  - **Output filename / binary property name:** `data` (this becomes the binary property referenced later for attachments)
- **Credentials:** HTMLCSS to PDF API credential (placeholder in JSON).
- **Connections:**
  - Input ← **Generate HTML Report**
  - Output → **Save to Google Drive** and **Send Compliance Report** (fan-out)
- **Version notes:** TypeVersion `1`
- **Edge cases / failures:**
  - External API downtime, invalid API key, rate limits.
  - HTML/CSS incompatibilities causing broken rendering.
  - Large HTML may exceed service limits.

---

### Block 1.4 — Delivery & Storage

**Overview:** Uses the PDF binary to send a formatted email via Gmail and archives the file in Google Drive.

**Nodes involved:**
- Send Compliance Report
- Save to Google Drive

#### Node: Send Compliance Report
- **Type / role:** `n8n-nodes-base.gmail` — Sends an email with the PDF attached.
- **Configuration (interpreted):**
  - **To:** `{{ $('Generate HTML Report').item.json.email }}`
  - **Subject:** `🔒 Website Compliance Report - {{ company_name }}`
  - **HTML message body:** Richly formatted summary:
    - Overall score with color-coded value
    - Quick summary table (privacy/cookies/terms/accessibility/ssl)
    - Conditional “Critical Issues” block rendered via expression
  - **Attachment:** binary property `data` (coming from **HTML to PDF** output) via attachmentsBinary.
- **Credentials:** Gmail OAuth2 credential (placeholder in JSON).
- **Connections:**
  - Input ← **HTML to PDF**
  - Output → none (terminal)
- **Version notes:** TypeVersion `2.2`
- **Edge cases / failures:**
  - Gmail OAuth scope/consent issues, expired refresh token.
  - Recipient email invalid or missing.
  - Attachment missing if PDF conversion fails or binary property name changes.

#### Node: Save to Google Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads the PDF to Drive for archiving.
- **Configuration (interpreted):**
  - **File name:** `Compliance_Report_{company_name}_{YYYY-MM-DD}.pdf`
  - **Drive:** “My Drive”
  - **Folder:** `root` (the cachedResultName suggests a custom folder in UI, but value is `root` in JSON)
  - **Binary upload source:** implied from previous node output; expects the PDF binary to be present (commonly `data`).
- **Credentials:** Google Drive OAuth2 credential (placeholder in JSON).
- **Connections:**
  - Input ← **HTML to PDF**
  - Output → none (terminal)
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - Wrong folder selection (root vs intended folder), permissions, shared drive nuances.
  - OAuth token expiry / missing scopes (`drive.file` vs `drive`).
  - If binary property is not detected automatically, upload may fail (depending on node configuration defaults).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry point: receive URL/email/company_name | — | Fetch Website HTML | ## Input & Website Scan<br>Webhook receives website URL, company name, and email. Fetches HTML content and cleans it for AI analysis. |
| Fetch Website HTML | n8n-nodes-base.httpRequest | Download website HTML | Webhook | Extract & Clean HTML | ## Input & Website Scan<br>Webhook receives website URL, company name, and email. Fetches HTML content and cleans it for AI analysis. |
| Extract & Clean HTML | n8n-nodes-base.code | Strip scripts/styles, normalize payload for AI | Fetch Website HTML | Analyze Compliance | ## Input & Website Scan<br>Webhook receives website URL, company name, and email. Fetches HTML content and cleans it for AI analysis. |
| Analyze Compliance | @n8n/n8n-nodes-langchain.openAi | LLM compliance audit returning JSON | Extract & Clean HTML | Parse Compliance Results | ## AI Compliance Analysis<br>OpenAI checks privacy policy, cookies, terms, SSL, accessibility, and contact info. Generates scores and identifies critical issues. |
| Parse Compliance Results | n8n-nodes-base.code | Parse JSON, fallback on errors, enrich metadata | Analyze Compliance | Generate HTML Report | ## AI Compliance Analysis<br>OpenAI checks privacy policy, cookies, terms, SSL, accessibility, and contact info. Generates scores and identifies critical issues. |
| Generate HTML Report | n8n-nodes-base.code | Build styled HTML report | Parse Compliance Results | HTML to PDF | ## Report Generation<br>Builds professional HTML compliance report with scores and recommendations, then converts to downloadable PDF. |
| HTML to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Convert HTML to PDF binary | Generate HTML Report | Save to Google Drive; Send Compliance Report | ## Report Generation<br>Builds professional HTML compliance report with scores and recommendations, then converts to downloadable PDF. |
| Send Compliance Report | n8n-nodes-base.gmail | Email PDF report to requester | HTML to PDF | — | ## Delivery & Storage<br>Emails PDF report to user and saves copy to Google Drive for auditing and legal review purposes. |
| Save to Google Drive | n8n-nodes-base.googleDrive | Archive PDF report in Drive | HTML to PDF | — | ## Delivery & Storage<br>Emails PDF report to user and saves copy to Google Drive for auditing and legal review purposes. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / operator notes | — | — | ### How it works<br>This workflow automatically audits websites for legal compliance. It receives a website URL via webhook, fetches and cleans the HTML content, uses OpenAI to analyze compliance across privacy policies, cookie consent, terms of service, accessibility (WCAG), SSL certificates, and contact information. The AI generates a comprehensive compliance report with scores and recommendations, converts it to PDF, emails it to the requester, and saves a copy to Google Drive for record-keeping.<br><br>### Setup steps<br>Connect these credentials before running:<br>* **OpenAI** - Analyze website compliance<br>* **HTMLCSS to PDF** - Convert reports to PDF<br>* **Gmail** - Send compliance reports<br>* **Google Drive** - Store PDF reports<br><br>### Customization<br>Modify AI compliance criteria in "Analyze Compliance" node system prompt. Customize HTML report design in "Generate HTML Report" node. Adjust PDF styling via HTMLCSS to PDF settings. Configure email templates in "Send Compliance Report" for branded messaging. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation: block label | — | — | ## Input & Website Scan<br>Webhook receives website URL, company name, and email. Fetches HTML content and cleans it for AI analysis. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation: block label | — | — | ## Report Generation<br>Builds professional HTML compliance report with scores and recommendations, then converts to downloadable PDF. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation: block label | — | — | ## AI Compliance Analysis<br>OpenAI checks privacy policy, cookies, terms, SSL, accessibility, and contact info. Generates scores and identifies critical issues. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation: block label | — | — | ## Delivery & Storage<br>Emails PDF report to user and saves copy to Google Drive for auditing and legal review purposes. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named **“Website Compliance Checker”**.

2. **Add a Webhook node**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `compliance-check`
   - Response: **Last node**
   - Save the workflow to get a test/production URL.

3. **Add an HTTP Request node** named **“Fetch Website HTML”**
   - URL: `{{ $json.body.url }}`
   - Response format: **Text**
   - Timeout: **30000 ms**
   - Keep “Allow Unauthorized Certs” off
   - Connect: **Webhook → Fetch Website HTML**

4. **Add a Code node** named **“Extract & Clean HTML”**
   - Paste logic that:
     - reads `$input.first().json.data` as HTML
     - strips `<script>` and `<style>` blocks
     - collapses whitespace
     - truncates to ~50,000 chars
     - outputs JSON with: `url`, `html_content`, `original_url`, `email`, `company_name` from the Webhook body
   - Connect: **Fetch Website HTML → Extract & Clean HTML**

5. **Add an OpenAI (LangChain) node** named **“Analyze Compliance”**
   - Credentials: configure **OpenAI API** credential in n8n and select it
   - Model: `chatgpt-4o-latest`
   - Temperature: **0.3**
   - Max tokens: **2000**
   - Messages:
     - System: define the auditor role + required output JSON schema (as in workflow)
     - User: include `{{ $json.url }}` and `{{ $json.html_content }}`
   - Connect: **Extract & Clean HTML → Analyze Compliance**

6. **Add a Code node** named **“Parse Compliance Results”**
   - Implement parsing that:
     - extracts the text content from the OpenAI output
     - strips ```json fences if present
     - `JSON.parse()` it
     - on failure returns a fallback schema and stores `raw_response`
     - enriches with `website_url`, `company_name`, `email`, `scan_date`, `scan_timestamp`, `has_error`
   - Connect: **Analyze Compliance → Parse Compliance Results**
   - Important: ensure the output path you parse matches your OpenAI node’s actual output structure; adjust extraction if needed.

7. **Add a Code node** named **“Generate HTML Report”**
   - Build an HTML string that:
     - uses the compliance fields and scores
     - conditionally renders critical issues and recommendations
     - outputs `{ ...data, html_report }`
   - Connect: **Parse Compliance Results → Generate HTML Report**

8. **Add an “HTML to PDF” node** named **“HTML to PDF”**
   - Credentials: create/select **HTMLCSS to PDF** API credential
   - HTML content: `{{ $json.html_report }}`
   - Output format: **File**
   - Output filename / binary property: `data`
   - Connect: **Generate HTML Report → HTML to PDF**

9. **Add a Gmail node** named **“Send Compliance Report”**
   - Credentials: create/select **Gmail OAuth2** credential (Google Cloud OAuth client + consent + scopes as required by n8n)
   - Operation: **Send**
   - To: `{{ $('Generate HTML Report').item.json.email }}`
   - Subject: `🔒 Website Compliance Report - {{ $('Generate HTML Report').item.json.company_name }}`
   - Message: set to **HTML** and paste your branded template (include expressions for scores)
   - Attachments (binary): attach property **`data`**
   - Connect: **HTML to PDF → Send Compliance Report**

10. **Add a Google Drive node** named **“Save to Google Drive”**
   - Credentials: create/select **Google Drive OAuth2** credential
   - Operation: **Upload**
   - File name: `Compliance_Report_{{ $('Generate HTML Report').item.json.company_name }}_{{ new Date().toISOString().split('T')[0] }}.pdf`
   - Drive: **My Drive**
   - Folder: choose target folder (or root)
   - Ensure it uploads the **binary PDF** produced by HTML to PDF (typically property `data`)
   - Connect: **HTML to PDF → Save to Google Drive**

11. **Test execution**
   - Call the webhook with JSON body, e.g.:
     - `url`: `https://example.com`
     - `email`: `recipient@domain.com`
     - `company_name`: `Example Co`
   - Verify:
     - OpenAI returns valid JSON (or parser fallback triggers)
     - PDF binary exists as `data`
     - Email sent with attachment
     - File appears in Drive

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically audits websites for legal compliance. It receives a website URL via webhook, fetches and cleans the HTML content, uses OpenAI to analyze compliance across privacy policies, cookie consent, terms of service, accessibility (WCAG), SSL certificates, and contact information. The AI generates a comprehensive compliance report with scores and recommendations, converts it to PDF, emails it to the requester, and saves a copy to Google Drive for record-keeping. | Sticky note: “How it works” |
| Connect these credentials before running: OpenAI, HTMLCSS to PDF, Gmail, Google Drive. | Sticky note: “Setup steps” |
| Customization: modify compliance criteria in “Analyze Compliance” system prompt; customize report design in “Generate HTML Report”; adjust PDF styling via HTMLCSS to PDF settings; configure email templates in “Send Compliance Report”. | Sticky note: “Customization” |