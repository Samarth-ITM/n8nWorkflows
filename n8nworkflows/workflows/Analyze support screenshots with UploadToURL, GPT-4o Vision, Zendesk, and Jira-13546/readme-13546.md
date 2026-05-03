Analyze support screenshots with UploadToURL, GPT-4o Vision, Zendesk, and Jira

https://n8nworkflows.xyz/workflows/analyze-support-screenshots-with-uploadtourl--gpt-4o-vision--zendesk--and-jira-13546


# Analyze support screenshots with UploadToURL, GPT-4o Vision, Zendesk, and Jira

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow ingests a customer support screenshot (or short screen recording), publishes it via **UploadToURL** to obtain a fast public CDN link, runs an **AI vision-style analysis** to extract developer-ready context (error text, affected component, repro steps, etc.), and then updates the corresponding **Zendesk ticket** or **Jira issue**. For high-severity cases, it escalates to **Slack**, and finally returns a structured JSON response to the caller.

**Target use cases:**
- Support teams receiving image/video attachments that lack technical detail
- Fast “support → dev” escalation with consistent formatting and severity mapping
- Standardized AI-enriched ticket comments and metadata tagging/labeling

### 1.1 Logical Blocks (by dependency)
1. **Entry, Validation & Enrichment**: Webhook → payload validation, normalization, structured filenames
2. **Upload / CDN Hosting**: choose remote URL vs binary upload → UploadToURL → normalize CDN URL
3. **AI Analysis & Severity Computation**: OpenAI node → parse JSON → compute/override severity → generate formatted comments
4. **Platform Routing & Ticket Updates**: route to Zendesk or Jira → update ticket/issue metadata → merge results
5. **Escalation + Final API Response**: IF severity triggers Slack → build final JSON → respond to webhook

> Important mismatch: The user-provided “Description” (HR/legal archiving with Google Drive + Airtable) does **not** match the actual workflow JSON, which implements **support screenshot analysis with UploadToURL + OpenAI + Zendesk/Jira + Slack**.

---

## 2. Block-by-Block Analysis

### Block A — Entry, Validation & Enrichment
**Overview:**  
Receives an inbound HTTP POST, validates required fields, normalizes ticket IDs per platform (Zendesk vs Jira), enforces an extension allowlist, and generates a structured filename plus metadata used downstream.

**Nodes involved:**
- **Webhook - Receive Screenshot**
- **Validate & Enrich Payload**
- **Has Remote URL?**

#### Node: Webhook - Receive Screenshot
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point
- **Configuration choices:**
  - Method: **POST**
  - Path: `visual-proof-ticket`
  - Response mode: **responseNode** (requires Respond to Webhook node to complete request)
  - Allowed origins: `*` (CORS permissive)
- **Inputs/Outputs:**
  - Output → **Validate & Enrich Payload**
- **Edge cases / failures:**
  - If caller sends multipart/binary, ensure webhook is configured in n8n UI to accept binary (and client uses correct form field names). The Code node currently checks `body.filename` rather than inspecting binary metadata.
  - Large uploads may hit n8n limits (body size, reverse proxy limits).

**Sticky note coverage:**  
- “Entry & Upload” note conceptually applies to this block (validation + upload behavior).

#### Node: Validate & Enrich Payload
- **Type / role:** `Code` — input normalization, validation, and metadata enrichment
- **Key logic/configuration:**
  - Reads: `const body = $input.first().json.body || $input.first().json;`
  - Valid platforms: `zendesk`, `jira` (defaults to `zendesk`)
  - Requires either:
    - `fileUrl` (remote asset), **or**
    - `filename` (intended for binary upload path)
  - Ticket ID validation:
    - Zendesk: numeric, supports `ZD-1234` prefix stripping
    - Jira: `PROJECT-123` uppercase
  - Extension allowlist: `png, jpg, jpeg, gif, webp, mp4, mov`
  - MIME type mapping; `isVideo` = `mp4|mov`
  - Sanitizes text fields (trim, max 500 chars)
  - Builds `structuredFilename`: `${ticketId}_screenshot_${YYYY-MM-DD}.${ext}`
  - Outputs unified JSON including: `platform, ticketId, cdnUrl (later), agent/customer info, productArea, userDescription, notifyDev, submittedAt`
- **Outputs:**
  - Output → **Has Remote URL?**
- **Edge cases / failures:**
  - If binary uploads are used, the workflow expects `body.filename`; many clients won’t send that in JSON body. In n8n, file metadata typically lives under `$binary` rather than `$json.body.filename`.
  - `fileUrl` extension inference may fail if URL has no extension or uses signed URLs without filename.
  - Sanitization truncates to 500 chars—fine for safety but may cut useful context.

**Sticky note coverage:**  
- “Entry & Upload” note applies.

#### Node: Has Remote URL?
- **Type / role:** `IF` — branching based on whether `fileUrl` is present
- **Condition:**
  - `{{ $json.fileUrl }}` **notEmpty**
- **Outputs:**
  - True → **Upload to URL - Remote**
  - False → **Upload to URL - Binary**
- **Edge cases:**
  - If both binary and `fileUrl` are provided, the remote path wins.

**Sticky note coverage:**  
- “Entry & Upload” note applies.

---

### Block B — Upload / CDN Hosting (UploadToURL)
**Overview:**  
Uploads the screenshot/video either from a remote URL or from binary data, then normalizes the UploadToURL response to a single `cdnUrl` field and related metadata.

**Nodes involved:**
- **Upload to URL - Remote**
- **Upload to URL - Binary**
- **Extract CDN URL**

#### Node: Upload to URL - Remote
- **Type / role:** Community node `n8n-nodes-uploadtourl.uploadToUrl` — uploads remote file to UploadToURL
- **Configuration:**
  - Operation: `uploadFile`
  - Credentials: “Upload to URL account 3”
- **Inputs/Outputs:**
  - Input from **Has Remote URL?** (true)
  - Output → **Extract CDN URL**
- **Edge cases / failures:**
  - If the community node requires an explicit field mapping to the remote URL, this workflow may be incomplete; the JSON shows only the operation, not the mapping.
  - UploadToURL API failures: auth errors (401/403), rate limits (429), size limits, timeouts.

**Sticky note coverage:**  
- “Entry & Upload” note applies.

#### Node: Upload to URL - Binary
- **Type / role:** Community node `n8n-nodes-uploadtourl.uploadToUrl` — uploads binary file to UploadToURL
- **Configuration:**
  - Operation: `uploadFile`
  - Credentials: “Upload to URL account 3”
- **Inputs/Outputs:**
  - Input from **Has Remote URL?** (false)
  - Output → **Extract CDN URL**
- **Edge cases / failures:**
  - As with remote, the node may require specifying which binary property to upload (e.g., `data`), not visible here.
  - Webhook must receive binary correctly; otherwise upload will fail.

**Sticky note coverage:**  
- “Entry & Upload” note applies.

#### Node: Extract CDN URL
- **Type / role:** `Code` — normalizes UploadToURL response, merges with validated metadata
- **Key logic:**
  - Reads upload response as `uploadResp`
  - Reads metadata from **Validate & Enrich Payload**
  - Attempts to extract URL from multiple shapes:
    - `uploadResp.url`, `uploadResp.link`, `uploadResp.data?.url`, `uploadResp.file?.url`, `uploadResp.shortUrl`
  - Forces HTTPS: `replace(/^http:\/\//, 'https://')`
  - Adds: `uploadId`, `fileSizeBytes`
- **Outputs:**
  - Output → **GPT-4o Vision - Analyse Screenshot**
- **Edge cases / failures:**
  - Throws hard error if no URL found, including truncated raw response (first 400 chars).
  - If UploadToURL changes response shape, extraction may break.

**Sticky note coverage:**  
- “Entry & Upload” note applies.

---

### Block C — AI Analysis & Severity Computation
**Overview:**  
Sends the CDN URL plus context fields to an OpenAI chat model, expects strict JSON back, parses it, then computes severity with an override keyword list. It also prebuilds formatted comments for Zendesk (Markdown) and Jira (wiki markup).

**Nodes involved:**
- **GPT-4o Vision - Analyse Screenshot**
- **Parse AI Analysis & Compute Severity**

#### Node: GPT-4o Vision - Analyse Screenshot
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat invocation
- **Configuration choices:**
  - Model is set to a fine-tuned `ft:gpt-3.5-turbo-0125:...` (as displayed in JSON), despite the node name claiming “GPT-4o Vision”.
  - `maxTokens: 1000`, `temperature: 0.3`
  - System message: instructs to return **ONLY valid JSON**, no markdown.
  - User message embeds:
    - `{{ $json.cdnUrl }}`, `{{ $json.productArea }}`, `{{ $json.userDescription }}`, `{{ $json.platform }}`, `{{ $json.ticketId }}`
  - Output is expected to be a JSON string in `message.content` or similar.
- **Inputs/Outputs:**
  - Input from **Extract CDN URL**
  - Output → **Parse AI Analysis & Compute Severity**
- **Version requirements:**
  - Node type version `1.5` suggests relatively recent n8n + langchain node bundle.
- **Edge cases / failures:**
  - **Vision capability mismatch:** The configured model appears to be a text-only fine-tuned GPT-3.5; it will not actually “see” images unless the node is configured to pass image inputs in the provider’s required format. Here it only passes a URL in text.
  - Model may return invalid JSON; downstream parsing will throw.

**Sticky note coverage:**  
- “AI Vision Analysis” note applies (with the caveat above).

#### Node: Parse AI Analysis & Compute Severity
- **Type / role:** `Code` — parses AI output, computes severity, builds comments
- **Key logic:**
  - Parses content from multiple likely output shapes:
    - `aiRaw.message?.content` or `aiRaw.choices[0].message.content` or `aiRaw.content` or `aiRaw.text`
  - Strict `JSON.parse`; throws error if invalid.
  - Severity override:
    - Critical keywords: `crash`, `data loss`, `payment failed`, `stripe`, `500`, `503`, `null pointer`, etc.
    - High keywords: `404`, `timeout`, `login failed`, etc.
  - Computes `severityBadge` mapping (includes emoji strings)
  - Builds:
    - `richComment` (Markdown) intended for Zendesk internal note
    - `jiraComment` (Jira wiki markup)
  - Outputs merged object with AI fields + computed severity + comment bodies
- **Outputs:**
  - Output → **Route by Platform**
- **Edge cases / failures:**
  - Any non-JSON response breaks workflow (no fallback behavior despite sticky note claiming graceful fallback).
  - Keyword-based overrides may over-escalate on ambiguous words (e.g., “timeout” in unrelated context).
  - `severityBadge` includes emoji; some ticket systems or corporate policies may dislike emoji in internal notes.

**Sticky note coverage:**  
- “AI Vision Analysis” note applies.

---

### Block D — Platform Routing & Ticket Updates (Zendesk/Jira)
**Overview:**  
Routes to Zendesk or Jira based on `platform`. Zendesk branch updates ticket tags; Jira branch updates summary/labels and then creates a comment. Both converge into a normalized response object.

**Nodes involved:**
- **Route by Platform**
- **Zendesk - Add Internal Note**
- **Jira - Update Issue Labels**
- **Jira - Add Comment**
- **Merge Platform Response**

> Sticky note mentions “Jira Attach File”, but **no such node exists** in the JSON. If raw file attachment is required, you must add it.

#### Node: Route by Platform
- **Type / role:** `Switch` — routes execution by platform
- **Rules:**
  - Output “Zendesk” if `{{ $json.platform }} == 'zendesk'`
  - Output “Jira” if `{{ $json.platform }} == 'jira'`
  - Fallback output: `extra` (unused)
- **Outputs:**
  - Zendesk → **Zendesk - Add Internal Note**
  - Jira → **Jira - Update Issue Labels**
- **Edge cases:**
  - If platform is invalid, it should already have been blocked earlier. If it slips through, it goes to fallback and then stops (no node connected).

**Sticky note coverage:**  
- “Platform Routing” note applies.

#### Node: Zendesk - Add Internal Note
- **Type / role:** `Zendesk` — updates a ticket
- **Configuration choices:**
  - Operation: `update`
  - Ticket id: `{{ $json.ticketId }}`
  - Updates tags: `visual-proof,ai-analysed,{{ $json.severity }}`
- **Inputs/Outputs:**
  - Input from **Route by Platform** (Zendesk)
  - Output → **Merge Platform Response**
- **Critical integration gap:**
  - Despite node name/notes claiming it “adds internal note”, the parameters shown only update **tags**. There is **no comment body** mapping to `richComment`.
  - To actually add an internal note, the Zendesk node must include a comment payload (typically `comment.body` and `comment.public=false`, depending on node capabilities).
- **Edge cases / failures:**
  - Zendesk credentials/permissions issues
  - Ticket not found (404)
  - Tag update success but missing comment undermines workflow’s stated purpose

**Sticky note coverage:**  
- “Platform Routing” note applies.

#### Node: Jira - Update Issue Labels
- **Type / role:** `Jira` — updates issue fields
- **Configuration:**
  - Issue key: `{{ $json.ticketId }}`
  - Operation: `update`
  - Labels: `{{ [...$json.suggestedLabels, 'visual-proof', $json.severity].join(',') }}`
  - Summary: `{{ $json.errorSummary }}`
- **Inputs/Outputs:**
  - Input from **Route by Platform** (Jira)
  - Output → **Jira - Add Comment**
- **Edge cases / failures:**
  - Jira label field expects an array in many APIs; the node may accept comma-separated string, but if not, this will fail.
  - Summary updates can be restricted by Jira permissions/screen configuration.
  - Issue key invalid or not found.

**Sticky note coverage:**  
- “Platform Routing” note applies.

#### Node: Jira - Add Comment
- **Type / role:** `Jira` — creates an issue comment
- **Configuration:**
  - Resource: `issueComment`
  - Operation: `create`
  - **Missing required fields** in the JSON snippet:
    - Typically needs `issueKey` and `comment`/`body` (e.g., set to `{{ $json.jiraComment }}`)
- **Inputs/Outputs:**
  - Input from **Jira - Update Issue Labels**
  - Output → **Merge Platform Response**
- **Edge cases / failures:**
  - Misconfiguration (missing required parameters) will prevent comment creation.
  - Jira markup vs Jira “ADF” format differences (depends on node/instance settings).

**Sticky note coverage:**  
- “Platform Routing” note applies.

#### Node: Merge Platform Response
- **Type / role:** `Code` — normalizes Zendesk/Jira branch results and builds a `ticketUrl`
- **Key expressions/variables:**
  - Uses `$vars.ZENDESK_SUBDOMAIN` and `$vars.JIRA_BASE_URL`
  - Ticket URL:
    - Zendesk: `https://{subdomain}.zendesk.com/agent/tickets/{ticketId}`
    - Jira: `{JIRA_BASE_URL}/browse/{ticketId}`
  - Extracts `platformCommentId` from various possible response shapes
- **Inputs/Outputs:**
  - Input from Zendesk or Jira branch
  - Output → **IF Critical or High?**
- **Edge cases / failures:**
  - If `$vars.*` are not defined, placeholder domains are used (`your-domain`), yielding invalid links.
  - `platformResp` shapes may differ from assumptions, resulting in `platformCommentId = null`.

**Sticky note coverage:**  
- “Platform Routing” note applies.

---

### Block E — Escalation + Final Response
**Overview:**  
If severity is not “medium” and notifications are enabled, it posts a Slack alert. Regardless, it builds a final structured JSON response and returns it to the webhook caller.

**Nodes involved:**
- **IF Critical or High?**
- **Slack - Escalate to Dev Channel**
- **Build Final Response**
- **Respond to Webhook**

#### Node: IF Critical or High?
- **Type / role:** `IF` — decides Slack escalation
- **Conditions (AND):**
  1. Severity is **not** equal to `medium`
  2. `notifyDev` is **true**
- **Important logic bug:**
  - This condition escalates for `low` as well (because `low != medium`).
  - If you truly want only `critical` and `high`, use “is in [critical, high]” or two equals checks combined with OR.
- **Outputs:**
  - True → **Slack - Escalate to Dev Channel**
  - False → **Build Final Response**
- **Edge cases:**
  - `severity` may be null if parsing fails earlier (but parsing currently hard-fails).

**Sticky note coverage:**  
- “Escalation & Response” note applies.

#### Node: Slack - Escalate to Dev Channel
- **Type / role:** `Slack` — posts escalation message
- **Configuration:**
  - Auth: OAuth2 credential “Mediajade Slack”
  - Message text uses:
    - `severityBadge`, `ticketUrl`, `ticketId`, `productArea`, customer/agent fields
    - `visibleErrorMessage`, `affectedComponent`, `browserOrOS`
    - `cdnUrl`, detected keywords and confidence score
- **Inputs/Outputs:**
  - Input from **IF Critical or High?** (true)
  - Output → **Build Final Response**
- **Edge cases / failures:**
  - Slack OAuth scopes missing (chat:write), channel access issues
  - Message formatting: includes `:jira:` and `:frame_with_picture:` tokens; these only render if Slack recognizes them.

**Sticky note coverage:**  
- “Escalation & Response” note applies.

#### Node: Build Final Response
- **Type / role:** `Code` — assembles stable API response for caller
- **Key logic:**
  - Reads Slack response as `$input.first()` (can be Slack output or, in the non-Slack path, the input from IF node).
  - Uses **Merge Platform Response** as source of truth for final data.
  - Returns a JSON object including:
    - `success`, `message`
    - `platform`, `ticketId`, `ticketUrl`, `platformCommentId`
    - `cdnUrl`, `structuredFilename`, `fileSizeBytes`
    - AI fields + computed severity
    - `devNotified` computed as `notifyDev && ['critical','high'].includes(severity)`
- **Inputs/Outputs:**
  - Input from either Slack node or IF false branch
  - Output → **Respond to Webhook**
- **Edge cases / failures:**
  - The variable `slackResp` is unused; harmless but confusing.
  - `devNotified` may be false even when Slack fired if the IF condition remains incorrect (e.g., “low” severity would Slack-fire, but `devNotified` would remain false).

**Sticky note coverage:**  
- “Escalation & Response” note applies.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — sends HTTP response back to caller
- **Configuration:**
  - Response code: `200`
  - Header: `Content-Type: application/json`
  - Body: `{{ $json }}` (entire JSON from previous node)
- **Inputs/Outputs:**
  - Input from **Build Final Response**
- **Edge cases:**
  - If any earlier node throws, caller receives an error (depending on n8n error handling settings).

**Sticky note coverage:**  
- “Escalation & Response” note applies.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | Sticky Note | Documentation / context | — | — | Analyze support screenshots with UploadToURL, OpenAI Vision, and Zendesk/Jira. Problem/solution summary; setup requirements; vars: ZENDESK_SUBDOMAIN, JIRA_BASE_URL. |
| Entry & Upload | Sticky Note | Documentation for entry+upload block | — | — | ## 🚪 Entry, Validation & Upload … supports fileUrl or binary; validates ticketId; allowlist extensions; UploadToURL; Extract CDN URL forces HTTPS. |
| AI Vision Analysis | Sticky Note | Documentation for AI block | — | — | ## 🤖 GPT-4o Vision Analysis … structured JSON fields; parse/validation; severity mapping/overrides. |
| Platform Routing | Sticky Note | Documentation for ticket platform updates | — | — | ## 🎫 Platform Routing — Zendesk & Jira … switch by platform; Zendesk internal note + tags; Jira comment + (mentions) attach raw file (but node missing). |
| Escalation & Response | Sticky Note | Documentation for escalation and webhook response | — | — | ## 📣 Severity Escalation & Response … Slack alert for critical/high; build final response; return 200 OK. |
| Webhook - Receive Screenshot | Webhook | Receives POST request + file metadata | — | Validate & Enrich Payload | Entry & Upload |
| Validate & Enrich Payload | Code | Validate inputs; normalize; build structured filename | Webhook - Receive Screenshot | Has Remote URL? | Entry & Upload |
| Has Remote URL? | IF | Choose remote vs binary upload path | Validate & Enrich Payload | Upload to URL - Remote; Upload to URL - Binary | Entry & Upload |
| Upload to URL - Remote | UploadToURL (community) | Upload remote `fileUrl` to CDN | Has Remote URL? (true) | Extract CDN URL | Entry & Upload |
| Upload to URL - Binary | UploadToURL (community) | Upload binary file to CDN | Has Remote URL? (false) | Extract CDN URL | Entry & Upload |
| Extract CDN URL | Code | Normalize UploadToURL response; enforce HTTPS | Upload to URL - Remote/Binary | GPT-4o Vision - Analyse Screenshot | Entry & Upload |
| GPT-4o Vision - Analyse Screenshot | OpenAI (LangChain) | Ask model to return JSON analysis | Extract CDN URL | Parse AI Analysis & Compute Severity | AI Vision Analysis |
| Parse AI Analysis & Compute Severity | Code | Parse AI JSON; compute severity; build comment bodies | GPT-4o Vision - Analyse Screenshot | Route by Platform | AI Vision Analysis |
| Route by Platform | Switch | Route to Zendesk vs Jira update path | Parse AI Analysis & Compute Severity | Zendesk - Add Internal Note; Jira - Update Issue Labels | Platform Routing |
| Zendesk - Add Internal Note | Zendesk | Update Zendesk ticket (tags; comment missing) | Route by Platform (Zendesk) | Merge Platform Response | Platform Routing |
| Jira - Update Issue Labels | Jira | Update summary + labels | Route by Platform (Jira) | Jira - Add Comment | Platform Routing |
| Jira - Add Comment | Jira | Create issue comment (fields missing) | Jira - Update Issue Labels | Merge Platform Response | Platform Routing |
| Merge Platform Response | Code | Normalize branch output; build ticketUrl | Zendesk - Add Internal Note / Jira - Add Comment | IF Critical or High? | Platform Routing |
| IF Critical or High? | IF | Decide Slack escalation | Merge Platform Response | Slack - Escalate to Dev Channel; Build Final Response | Escalation & Response |
| Slack - Escalate to Dev Channel | Slack | Post high-severity alert to dev channel | IF Critical or High? (true) | Build Final Response | Escalation & Response |
| Build Final Response | Code | Assemble final JSON result | Slack - Escalate to Dev Channel OR IF false branch | Respond to Webhook | Escalation & Response |
| Respond to Webhook | Respond to Webhook | Return 200 JSON to caller | Build Final Response | — | Escalation & Response |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name it: *Analyze support screenshots with UploadToURL, GPT-4o Vision, Zendesk, and Jira* (or your preferred name).

2. **Add node: Webhook**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `visual-proof-ticket`
   - Response mode: **Using “Respond to Webhook” node** (Response Mode = `responseNode`)
   - (Optional) Configure binary upload support in webhook settings if you want multipart file uploads.

3. **Add node: Code — “Validate & Enrich Payload”**
   - Paste logic equivalent to:
     - Validate `platform` in `zendesk|jira` (default `zendesk`)
     - Require `ticketId`
     - Require either `fileUrl` or `filename` (for binary path)
     - Allowlist extensions: png/jpg/jpeg/gif/webp/mp4/mov
     - Build `structuredFilename` and set `mimeType`, `isVideo`
     - Sanitize strings, keep `notifyDev` default true
   - Connect: **Webhook → Validate & Enrich Payload**

4. **Add node: IF — “Has Remote URL?”**
   - Condition: String `{{ $json.fileUrl }}` **is not empty**
   - Connect: **Validate & Enrich Payload → Has Remote URL?**

5. **Install and configure UploadToURL community node**
   - Install community node: `n8n-nodes-uploadtourl`
   - Create credentials for UploadToURL API (per node docs / your UploadToURL account)

6. **Add node: UploadToURL — “Upload to URL - Remote”**
   - Operation: **Upload File**
   - Configure it to upload **from remote URL** (map the URL field to `{{$json.fileUrl}}` if required by the node UI)
   - Connect: **Has Remote URL? (true) → Upload to URL - Remote**

7. **Add node: UploadToURL — “Upload to URL - Binary”**
   - Operation: **Upload File**
   - Configure it to upload **from binary data** (select the binary property name used by your webhook, often `data`)
   - Connect: **Has Remote URL? (false) → Upload to URL - Binary**

8. **Add node: Code — “Extract CDN URL”**
   - Implement:
     - Merge upload response + validated metadata
     - Extract public URL from several response fields
     - Force https
     - Output: `cdnUrl`, `uploadId`, `fileSizeBytes`
   - Connect:
     - **Upload to URL - Remote → Extract CDN URL**
     - **Upload to URL - Binary → Extract CDN URL**

9. **Add node: OpenAI (LangChain) — “GPT-4o Vision - Analyse Screenshot”**
   - Credentials: create/select **OpenAI API** credential
   - Model:
     - If you truly need *vision*, choose a **vision-capable model** (e.g., GPT-4o) and configure the node to send an image input as supported by your n8n OpenAI node version.
     - If using URL-in-text only, understand it may not actually analyze pixels.
   - Prompt:
     - System: “Return ONLY valid JSON”
     - User: include `{{$json.cdnUrl}}`, product area, description, platform, ticketId
     - Ask for the JSON schema used downstream (errorSummary, visibleErrorMessage, errorCode, etc.)
   - Connect: **Extract CDN URL → GPT node**

10. **Add node: Code — “Parse AI Analysis & Compute Severity”**
    - Parse AI output with `JSON.parse`
    - Compute severity with keyword overrides
    - Build:
      - `richComment` (Markdown for Zendesk)
      - `jiraComment` (Jira wiki markup or ADF depending on your Jira configuration)
    - Connect: **GPT node → Parse AI Analysis & Compute Severity**

11. **Add node: Switch — “Route by Platform”**
    - Rule 1: if `{{$json.platform}} == zendesk` → Output “Zendesk”
    - Rule 2: if `{{$json.platform}} == jira` → Output “Jira”
    - Connect: **Parse AI Analysis & Compute Severity → Route by Platform**

12. **Zendesk branch**
    1. Add node: **Zendesk** — “Zendesk - Add Internal Note”
       - Credentials: Zendesk API credentials (API token/OAuth depending on node)
       - Operation: **Update Ticket**
       - Ticket ID: `{{$json.ticketId}}`
       - Add tags: `visual-proof`, `ai-analysed`, `{{$json.severity}}`
       - **Also add an internal comment** using `{{$json.richComment}}` (this is missing in the provided JSON; configure it in the node UI if supported).
    2. Connect: **Route by Platform (Zendesk) → Zendesk node**

13. **Jira branch**
    1. Add node: **Jira** — “Jira - Update Issue Labels”
       - Credentials: Jira (Cloud) OAuth2 / API token
       - Issue key: `{{$json.ticketId}}`
       - Update summary to `{{$json.errorSummary}}`
       - Update labels to include suggested labels + `visual-proof` + severity  
         (Ensure correct data type: array vs comma-separated string depending on node.)
    2. Add node: **Jira** — “Jira - Add Comment”
       - Resource: Issue Comment
       - Operation: Create
       - Issue key: `{{$json.ticketId}}`
       - Comment body: `{{$json.jiraComment}}` (or ADF equivalent)
    3. Connect:  
       - **Route by Platform (Jira) → Jira - Update Issue Labels → Jira - Add Comment**

    > Optional (mentioned by sticky note but absent in JSON): add a **Jira attachment** node/call to attach the raw file.

14. **Add node: Code — “Merge Platform Response”**
    - Build `ticketUrl` using workflow variables:
      - `ZENDESK_SUBDOMAIN`
      - `JIRA_BASE_URL`
    - Normalize returned `platformCommentId` from Zendesk/Jira responses
    - Connect:
      - **Zendesk node → Merge Platform Response**
      - **Jira - Add Comment → Merge Platform Response**

15. **Add node: IF — “IF Critical or High?”**
    - Recommended condition (fixing the current logic):
      - `{{$json.severity}}` is in `critical, high`
      - AND `{{$json.notifyDev}}` is true
    - Connect: **Merge Platform Response → IF Critical or High?**

16. **Add node: Slack — “Slack - Escalate to Dev Channel”**
    - Credentials: Slack OAuth2
    - Post message using fields from the merged payload (ticket URL, severity badge, cdnUrl, etc.)
    - Connect: **IF true → Slack**

17. **Add node: Code — “Build Final Response”**
    - Return final JSON with:
      - ticket details, cdnUrl, severity, AI fields, timestamps, `devNotified`
    - Connect:
      - **Slack → Build Final Response**
      - **IF false → Build Final Response**

18. **Add node: Respond to Webhook**
    - Response code: 200
    - Content-Type: application/json
    - Body: `{{$json}}`
    - Connect: **Build Final Response → Respond to Webhook**

19. **Set workflow variables**
    - `ZENDESK_SUBDOMAIN` (e.g., `acme`)
    - `JIRA_BASE_URL` (e.g., `https://acme.atlassian.net`)

20. **Test**
    - Send POST JSON with remote URL:
      - `platform`, `ticketId`, `fileUrl`, `description`, `productArea`, customer/agent fields
    - Verify:
      - UploadToURL returns a public link
      - AI node returns valid JSON
      - Ticket updated correctly
      - Slack fires only for critical/high (after fixing IF)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| UploadToURL community node required: `n8n-nodes-uploadtourl` | Install via n8n Community Nodes; required for both Upload nodes. |
| Workflow variables required | `ZENDESK_SUBDOMAIN` and `JIRA_BASE_URL` are used to build ticket URLs in “Merge Platform Response”. |
| Sticky note claims Jira file attachment step | The note mentions “Jira Attach File”, but no such node exists in the provided JSON—add it if you need raw attachment in Jira. |
| AI node naming vs configured model mismatch | Node is named “GPT-4o Vision” but is configured with a fine-tuned GPT-3.5 model ID; vision analysis may not actually occur without a vision-capable configuration. |
| HR/legal archiving description mismatch | The provided prose describes a different workflow (Google Drive + Airtable + email confirmations). The JSON implements a support screenshot pipeline (UploadToURL + OpenAI + Zendesk/Jira + Slack). |

