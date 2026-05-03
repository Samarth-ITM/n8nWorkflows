Generate branded PDF reports from incoming emails using Autype and OpenRouter

https://n8nworkflows.xyz/workflows/generate-branded-pdf-reports-from-incoming-emails-using-autype-and-openrouter-14184


# Generate branded PDF reports from incoming emails using Autype and OpenRouter

# 1. Workflow Overview

This workflow monitors an email inbox for incoming document requests, optionally processes attached PDF files through OCR, asks an AI agent to generate a polished document in Autype Extended Markdown, renders that markdown into a branded PDF, and emails the finished PDF back to the sender.

Typical use cases include:
- Summarizing one or more attached PDF documents
- Comparing multiple attached PDFs
- Rewriting or drafting a polished document from incoming instructions
- Creating a new document from scratch, optionally using web research tools
- Delivering the result as a company-branded PDF

The workflow is organized into four main logical blocks.

## 1.1 Email Input & Branding Configuration

The workflow starts from an IMAP email trigger. It downloads attachments, extracts sender and request text, and defines reusable company branding variables such as company name, logo URL, and brand color.

## 1.2 PDF Detection and OCR Processing

If PDF attachments are present, the workflow splits them into separate items, loops through them one by one, uploads each file to Autype, starts OCR with Autype Lens via HTTP Request nodes, waits, polls OCR job status, extracts OCR text, and finally combines all OCR results into one consolidated context.

If no PDFs are attached, the workflow bypasses OCR and prepares a text-only context.

## 1.3 AI Document Generation

The workflow downloads the Autype markdown syntax reference, merges it with the request context, then sends everything to an AI agent. The AI agent uses an OpenRouter chat model and may optionally call SerpAPI or Firecrawl when external research is explicitly needed.

## 1.4 PDF Rendering and Email Delivery

The AI output is cleaned, transformed into a rendering payload, rendered as a branded PDF through Autype, and then sent back to the original email sender through SMTP with the generated PDF attached.

---

# 2. Block-by-Block Analysis

## Block 1 — Email Input & Branding Configuration

### Overview

This block receives incoming emails, downloads attachments, defines company branding variables, and converts the email into a normalized request structure. It is the entry point and prepares all downstream context.

### Nodes Involved

- New Email Received
- Set Company Config
- Extract & Split PDFs
- Has PDFs?

### Node Details

#### 1. New Email Received

- **Type and technical role:** `n8n-nodes-base.emailReadImap`  
  IMAP trigger node that watches a mailbox for new emails.
- **Configuration choices:**
  - Attachment download is enabled.
  - No extra trigger filters are configured in the JSON shown.
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to: `Set Company Config`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - IMAP authentication failure
  - Mailbox connectivity or TLS issues
  - Large attachments causing memory or timeout problems
  - Unexpected email formats with missing `from`, `subject`, or body fields
- **Sub-workflow reference:** None

#### 2. Set Company Config

- **Type and technical role:** `n8n-nodes-base.set`  
  Adds reusable branding/configuration fields to the current item.
- **Configuration choices:**
  - Keeps existing fields (`includeOtherFields: true`)
  - Adds:
    - `companyName = "Your Company Name"`
    - `companyLogoUrl = "https://placehold.co/50x50/orange/white"`
    - `brandColor = "#1a5276"`
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**
  - Input from: `New Email Received`
  - Output to: `Extract & Split PDFs`
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Misconfigured branding values can degrade PDF rendering quality
  - Invalid image URL for logo may lead to missing header image in final PDF
- **Sub-workflow reference:** None

#### 3. Extract & Split PDFs

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses email metadata and attachments, extracts request text, detects PDF files, and emits either:
  - one item per PDF attachment, or
  - one text-only item when no PDFs are found
- **Configuration choices:**
  - Reads email fields from `$input.first()`
  - Builds:
    - `requestText` from subject + body
    - `senderEmail` from parsed `from`
    - `senderName` from parsed `from`
  - Scans `binary` attachments for PDFs by MIME type or filename extension
  - Truncates email body to 5000 characters
  - Renames per-PDF binary to `data` for easier downstream handling
- **Key expressions or variables used:**
  - `emailData.subject`
  - `emailData.textPlain || emailData.text || emailData.html`
  - Regex for sender email extraction: `/\<([^>]+)\>/`
  - `binary[key].mimeType`
  - `binary[key].fileName`
- **Input and output connections:**
  - Input from: `Set Company Config`
  - Output to: `Has PDFs?`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - HTML-only emails may produce noisy request text
  - Very large email body is truncated
  - Sender parsing may fail for uncommon `from` formats
  - Attachments without MIME metadata rely on filename extension detection
  - Non-PDF attachments are ignored
- **Sub-workflow reference:** None

#### 4. Has PDFs?

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches the workflow depending on whether PDF attachments were detected.
- **Configuration choices:**
  - Checks boolean condition: `{{$json.hasPdfs}} is true`
- **Key expressions or variables used:**
  - `={{ $json.hasPdfs }}`
- **Input and output connections:**
  - Input from: `Extract & Split PDFs`
  - True branch to: `Loop Over PDFs`
  - False branch to: `Prepare Text Only`
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - If `hasPdfs` is missing or not a strict boolean, branching may behave unexpectedly
- **Sub-workflow reference:** None

---

## Block 2 — PDF Detection and OCR Processing

### Overview

This block handles attached PDFs sequentially. Each PDF is uploaded to Autype, OCR is started through the Lens API, the workflow waits before polling status, extracted text is normalized, and all OCR output is consolidated into a single context string.

### Nodes Involved

- Loop Over PDFs
- Upload PDF to Autype
- Autype Lens OCR
- Wait for OCR
- Poll OCR Status
- Extract OCR Text
- Combine All OCR Results
- Prepare Text Only

### Node Details

#### 5. Loop Over PDFs

- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through PDF items one at a time.
- **Configuration choices:**
  - Uses default batch behavior
  - `reset: false`
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input from true branch of: `Has PDFs?`
  - Batch output to: `Upload PDF to Autype`
  - Completion/output path to: `Combine All OCR Results`
  - Receives loop-back input from: `Extract OCR Text`
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**
  - Large numbers of PDFs may significantly increase runtime
  - If one item fails hard and error handling is not customized, the loop may stop
- **Sub-workflow reference:** None

#### 6. Upload PDF to Autype

- **Type and technical role:** `n8n-nodes-autype.autype`  
  Uploads the current PDF file to Autype as a file resource.
- **Configuration choices:**
  - Resource: `file`
  - Uses Autype credentials
  - The node likely relies on the incoming binary `data` generated in the code node
- **Key expressions or variables used:** None explicit in parameters.
- **Input and output connections:**
  - Input from: `Loop Over PDFs`
  - Output to: `Autype Lens OCR`
- **Version-specific requirements:**
  - Community node `n8n-nodes-autype` must be installed
  - Only available on self-hosted n8n
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Missing community node installation
  - Autype authentication failure
  - File upload size limitations
  - Missing or malformed incoming binary
- **Sub-workflow reference:** None

#### 7. Autype Lens OCR

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Starts an OCR job on the uploaded Autype file via Autype Lens API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.autype.com/api/v1/dev/tools/lens/ocr`
  - JSON body:
    - `fileId = {{$json.id}}`
    - `outputFormat = "md"`
  - Authentication: Generic credential type using HTTP Header Auth
- **Key expressions or variables used:**
  - `{{ $json.id }}`
- **Input and output connections:**
  - Input from: `Upload PDF to Autype`
  - Output to: `Wait for OCR`
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - Incorrect header auth credential
  - API key missing or invalid
  - File ID missing if upload failed or returned unexpected payload
  - API schema changes could break request body expectations
- **Sub-workflow reference:** None

#### 8. Wait for OCR

- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces a delay before checking OCR job status.
- **Configuration choices:**
  - Wait amount: `8`
  - Unit is not explicitly shown in the visible JSON parameters; in n8n wait nodes this usually depends on node defaults/UI interpretation
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from: `Autype Lens OCR`
  - Output to: `Poll OCR Status`
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Delay may be too short for larger PDFs
  - Excessively long delay slows throughput
- **Sub-workflow reference:** None

#### 9. Poll OCR Status

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the OCR job status from Autype.
- **Configuration choices:**
  - URL uses OCR job ID from `Autype Lens OCR`
  - Method defaults to GET
  - Authentication: Generic credential type using HTTP Header Auth
- **Key expressions or variables used:**
  - `=https://api.autype.com/api/v1/dev/tools/jobs/{{ $('Autype Lens OCR').item.json.id }}`
- **Input and output connections:**
  - Input from: `Wait for OCR`
  - Output to: `Extract OCR Text`
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - Job ID lookup failure
  - API auth error
  - OCR still pending when polled
  - Network timeout
- **Sub-workflow reference:** None

#### 10. Extract OCR Text

- **Type and technical role:** `n8n-nodes-base.code`  
  Converts OCR status payload into a normalized text result for each PDF.
- **Configuration choices:**
  - Reads current OCR status
  - Looks up the current loop item filename from `Loop Over PDFs`
  - If completed, uses markdown/text fields from result or metadata
  - Truncates OCR text to 15000 characters
  - Returns placeholder text for failed or still-processing jobs
- **Key expressions or variables used:**
  - `item.json.status`
  - `$('Loop Over PDFs').first().json`
  - `item.json.result?.markdown`
  - `item.json.result?.text`
  - `item.json.metadata?.markdown`
  - `item.json.metadata?.text`
- **Input and output connections:**
  - Input from: `Poll OCR Status`
  - Output back to: `Loop Over PDFs`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Status may not be `COMPLETED` after one poll
  - Returned OCR fields may differ from expected schema
  - Filename lookup from loop context could become fragile if flow is changed
- **Sub-workflow reference:** None

#### 11. Combine All OCR Results

- **Type and technical role:** `n8n-nodes-base.code`  
  Collects all OCR outputs from loop execution and merges them into a single `pdfContent` string.
- **Configuration choices:**
  - Reads all items from `Extract OCR Text`
  - Reads original request data from `Extract & Split PDFs`
  - Builds a markdown-formatted combined section for each PDF
- **Key expressions or variables used:**
  - `$('Extract OCR Text').all()`
  - `$('Extract & Split PDFs').first().json`
- **Input and output connections:**
  - Input from completion path of: `Loop Over PDFs`
  - Output to: `Download Markdown Syntax`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If loop does not produce items, combined output may be empty
  - If OCR placeholders are returned, the AI will use them as source text
- **Sub-workflow reference:** None

#### 12. Prepare Text Only

- **Type and technical role:** `n8n-nodes-base.code`  
  Builds a context object when the email has no PDF attachments.
- **Configuration choices:**
  - Sets:
    - `pdfContent = ''`
    - `pdfCount = 0`
    - `hasPdfs = false`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input from false branch of: `Has PDFs?`
  - Output to: `Download Markdown Syntax`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If upstream email parsing failed, request context may be incomplete
- **Sub-workflow reference:** None

---

## Block 3 — AI Document Generation

### Overview

This block fetches the Autype markdown syntax reference, merges it with the user request and OCR context, and invokes an AI agent. The agent uses OpenRouter as the language model and can optionally call SerpAPI or Firecrawl for research.

### Nodes Involved

- Download Markdown Syntax
- Merge Context
- AI Document Assistant
- OpenRouter Chat Model
- SerpAPI
- /scrape in Firecrawl

### Node Details

#### 13. Download Markdown Syntax

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the Autype Extended Markdown syntax reference as plain text.
- **Configuration choices:**
  - URL: `https://autype.com/llm-resources/markdown-syntax.md`
  - Response format: text
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from:
    - `Prepare Text Only`, or
    - `Combine All OCR Results`
  - Output to: `Merge Context`
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - Remote document unavailable
  - HTTP timeout or connectivity failure
  - Syntax file format changes
- **Sub-workflow reference:** None

#### 14. Merge Context

- **Type and technical role:** `n8n-nodes-base.code`  
  Merges markdown syntax text with whichever context branch executed.
- **Configuration choices:**
  - Reads downloaded syntax from current input
  - Tries context in this order:
    1. `Combine All OCR Results`
    2. `Prepare Text Only`
    3. fallback `Extract & Split PDFs`
  - Truncates syntax reference to 12000 characters
- **Key expressions or variables used:**
  - `$input.first().json?.data`
  - `$('Combine All OCR Results').first().json`
  - `$('Prepare Text Only').first().json`
  - `$('Extract & Split PDFs').first().json`
- **Input and output connections:**
  - Input from: `Download Markdown Syntax`
  - Output to: `AI Document Assistant`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If node names change, referenced lookups will break
  - If syntax download returns unexpected structure, `data` may be empty
- **Sub-workflow reference:** None

#### 15. AI Document Assistant

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI agent that interprets the request and generates Autype Extended Markdown.
- **Configuration choices:**
  - Prompt is explicitly defined
  - Main prompt includes:
    - request text
    - optional OCR content
    - instructions for summarize/compare/draft/create-from-scratch behavior
    - tool usage rules
    - output formatting rules
  - System message includes:
    - capability definition
    - hard constraints on tool usage
    - markdown syntax reference
    - structural formatting requirements
  - `maxIterations = 6`
- **Key expressions or variables used:**
  - `{{ $json.requestText }}`
  - conditional inclusion of OCR content via `{{ $json.hasPdfs ? ... : ... }}`
  - `{{ $json.markdownSyntax || 'Standard Markdown with extended features' }}`
- **Input and output connections:**
  - Main input from: `Merge Context`
  - AI language model input from: `OpenRouter Chat Model`
  - AI tool inputs from:
    - `SerpAPI`
    - `/scrape in Firecrawl`
  - Main output to: `Prepare Render Payload`
- **Version-specific requirements:**
  - Requires LangChain nodes package supported by current n8n build
  - `typeVersion: 2`
- **Edge cases or potential failure types:**
  - LLM auth or quota errors
  - Tool call failures from SerpAPI or Firecrawl
  - Model may still return code fences despite instruction
  - Very large OCR input may exceed context limits depending on selected model
  - Agent behavior depends on model reasoning quality
- **Sub-workflow reference:** None

#### 16. OpenRouter Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Supplies the chat LLM used by the AI agent.
- **Configuration choices:**
  - Model: `z-ai/glm-5`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI language model connection to: `AI Document Assistant`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - OpenRouter auth failure
  - Unsupported or deprecated model name
  - Model-specific token/context limitations
- **Sub-workflow reference:** None

#### 17. SerpAPI

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolSerpApi`  
  Search tool exposed to the AI agent for web research.
- **Configuration choices:**
  - No special options configured
- **Key expressions or variables used:** None directly; agent controls the query.
- **Input and output connections:**
  - AI tool connection to: `AI Document Assistant`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - SerpAPI quota/auth issues
  - Search results may be noisy or irrelevant
  - Agent may misuse the tool unless prompt constraints are followed
- **Sub-workflow reference:** None

#### 18. /scrape in Firecrawl

- **Type and technical role:** `@mendable/n8n-nodes-firecrawl.firecrawlTool`  
  Scraping tool exposed to the AI agent for fetching page content from a known URL.
- **Configuration choices:**
  - Operation: `scrape`
  - URL is filled dynamically via `$fromAI(...)`
- **Key expressions or variables used:**
  - `={{ $fromAI('URL', ``, 'string') }}`
- **Input and output connections:**
  - AI tool connection to: `AI Document Assistant`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Firecrawl auth failure
  - Site blocking or scrape failure
  - Bad URL inferred by the model
- **Sub-workflow reference:** None

---

## Block 4 — PDF Rendering and Email Delivery

### Overview

This block cleans the AI-generated markdown, prepares file metadata and branding values, renders the markdown into a branded PDF through Autype, and sends the PDF back to the sender via SMTP.

### Nodes Involved

- Prepare Render Payload
- Render Branded PDF
- Send Report via Email

### Node Details

#### 19. Prepare Render Payload

- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes the AI output and constructs the payload required for PDF rendering.
- **Configuration choices:**
  - Reads agent output from `output` or `text`
  - Removes markdown code fences if present
  - Builds a sanitized filename from request text
  - Adds current date to filename
  - Pulls branding data from `Set Company Config`
  - Pulls sender email from `Merge Context`
- **Key expressions or variables used:**
  - `$json.output || $json.text`
  - `$('Set Company Config').first().json`
  - `$('Merge Context').first().json`
- **Input and output connections:**
  - Input from: `AI Document Assistant`
  - Output to: `Render Branded PDF`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Empty AI output results in blank PDF
  - Filename sanitization may produce very short or generic names
  - Node-name dependencies are fragile if renamed
- **Sub-workflow reference:** None

#### 20. Render Branded PDF

- **Type and technical role:** `n8n-nodes-autype.autype`  
  Renders Autype Extended Markdown into a downloadable branded PDF.
- **Configuration choices:**
  - Operation: `renderMarkdown`
  - Downloads output binary
  - Content comes from `{{$json.markdown}}`
  - Paper size: `A4`
  - Orientation: `portrait`
  - Title from `{{$json.title}}`
  - Defaults JSON defines:
    - font family and font size
    - line height
    - heading numbering
    - spacing rules
    - chart colors
    - heading styles
    - table styling
    - header with logo/company name
    - footer with company name and page numbering
- **Key expressions or variables used:**
  - `={{ $json.markdown }}`
  - `={{ $json.title }}`
  - `={{ JSON.stringify({...}) }}`
  - Uses `$json.brandColor`, `$json.companyLogoUrl`, `$json.companyName`
- **Input and output connections:**
  - Input from: `Prepare Render Payload`
  - Output to: `Send Report via Email`
- **Version-specific requirements:**
  - Requires `n8n-nodes-autype` community node
  - Self-hosted n8n required
  - `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Invalid markdown may render poorly
  - Invalid logo URL can affect header rendering
  - JSON defaults expression could fail if malformed during edits
  - Autype API limits or auth errors
- **Sub-workflow reference:** None

#### 21. Send Report via Email

- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends the generated PDF as an email attachment.
- **Configuration choices:**
  - Attachment source: binary property `data`
  - Subject: `Your Document: {{ $('Prepare Render Payload').item.json.title }}`
  - Recipient: original email sender from `New Email Received`
  - Sender address is statically configured
- **Key expressions or variables used:**
  - `={{ $('New Email Received').item.json.from }}`
  - `=Your Document: {{ $('Prepare Render Payload').item.json.title }}`
  - `=Your Name <your-email@example.com>`
- **Input and output connections:**
  - Input from: `Render Branded PDF`
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases or potential failure types:**
  - SMTP authentication or relay issues
  - Some SMTP providers reject spoofed/mismatched `fromEmail`
  - Using the raw `from` header instead of parsed email may fail with some providers
  - If Autype binary output property is not `data`, attachment may be missing
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Email Received | n8n-nodes-base.emailReadImap | IMAP trigger for incoming document request emails |  | Set Company Config | ### 1. Email Input & Config\nThe IMAP Email Trigger monitors your inbox for incoming document requests. PDF attachments are automatically detected. The Set Company Config node defines branding variables. |
| Set Company Config | n8n-nodes-base.set | Adds branding/config values used later in rendering | New Email Received | Extract & Split PDFs | ### 1. Email Input & Config\nThe IMAP Email Trigger monitors your inbox for incoming document requests. PDF attachments are automatically detected. The Set Company Config node defines branding variables. |
| Extract & Split PDFs | n8n-nodes-base.code | Extracts email text and splits PDF attachments into per-file items | Set Company Config | Has PDFs? | ### 1. Email Input & Config\nThe IMAP Email Trigger monitors your inbox for incoming document requests. PDF attachments are automatically detected. The Set Company Config node defines branding variables. |
| Has PDFs? | n8n-nodes-base.if | Branches between OCR path and text-only path | Extract & Split PDFs | Loop Over PDFs; Prepare Text Only | ### 1. Email Input & Config\nThe IMAP Email Trigger monitors your inbox for incoming document requests. PDF attachments are automatically detected. The Set Company Config node defines branding variables. |
| Loop Over PDFs | n8n-nodes-base.splitInBatches | Iterates through PDF attachments one at a time | Has PDFs?; Extract OCR Text | Upload PDF to Autype; Combine All OCR Results | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Upload PDF to Autype | n8n-nodes-autype.autype | Uploads each PDF file to Autype | Loop Over PDFs | Autype Lens OCR | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Autype Lens OCR | n8n-nodes-base.httpRequest | Starts OCR job for uploaded PDF | Upload PDF to Autype | Wait for OCR | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Wait for OCR | n8n-nodes-base.wait | Delays before polling OCR job status | Autype Lens OCR | Poll OCR Status | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Poll OCR Status | n8n-nodes-base.httpRequest | Checks OCR job state in Autype | Wait for OCR | Extract OCR Text | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Extract OCR Text | n8n-nodes-base.code | Normalizes OCR job result into text per PDF | Poll OCR Status | Loop Over PDFs | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Combine All OCR Results | n8n-nodes-base.code | Merges all OCR outputs into one context block | Loop Over PDFs | Download Markdown Syntax | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Prepare Text Only | n8n-nodes-base.code | Builds request context when no PDFs are attached | Has PDFs? | Download Markdown Syntax |  |
| Download Markdown Syntax | n8n-nodes-base.httpRequest | Downloads Autype markdown syntax reference | Prepare Text Only; Combine All OCR Results | Merge Context | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| Merge Context | n8n-nodes-base.code | Combines request context with markdown syntax reference | Download Markdown Syntax | AI Document Assistant | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| AI Document Assistant | @n8n/n8n-nodes-langchain.agent | Generates the markdown document and optionally uses research tools | Merge Context; OpenRouter Chat Model; SerpAPI; /scrape in Firecrawl | Prepare Render Payload | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides the LLM backing the AI agent |  | AI Document Assistant | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| SerpAPI | @n8n/n8n-nodes-langchain.toolSerpApi | Search tool for AI research |  | AI Document Assistant | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| /scrape in Firecrawl | @mendable/n8n-nodes-firecrawl.firecrawlTool | URL scraping tool for AI research |  | AI Document Assistant | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| Prepare Render Payload | n8n-nodes-base.code | Cleans AI markdown and prepares rendering metadata | AI Document Assistant | Render Branded PDF | ### 4. Render & Send\nThe AI output is cleaned and rendered to a branded PDF via Autype Render from Markdown. Company styling is applied via a defaults JSON (fonts, colors, heading styles, table styles, header with logo, footer with page numbers). The PDF is sent back to the sender via SMTP. |
| Render Branded PDF | n8n-nodes-autype.autype | Renders branded PDF from markdown | Prepare Render Payload | Send Report via Email | ### 4. Render & Send\nThe AI output is cleaned and rendered to a branded PDF via Autype Render from Markdown. Company styling is applied via a defaults JSON (fonts, colors, heading styles, table styles, header with logo, footer with page numbers). The PDF is sent back to the sender via SMTP. |
| Send Report via Email | n8n-nodes-base.emailSend | Emails final PDF back to sender | Render Branded PDF |  | ### 4. Render & Send\nThe AI output is cleaned and rendered to a branded PDF via Autype Render from Markdown. Company styling is applied via a defaults JSON (fonts, colors, heading styles, table styles, header with logo, footer with page numbers). The PDF is sent back to the sender via SMTP. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / overview note |  |  | ## AI Document Assistant: Branded PDF Generation from Email\n\nReceive an email with a document request and optional PDF attachments. The AI assistant can create drafts, summarize documents, compare multiple PDFs, or write new documents with internet research — all output as professionally branded PDFs using Autype.\n\n### How it works\n1. **Email Trigger** — Monitors inbox for incoming document requests with optional PDF attachments.\n2. **Extract & Split PDFs** — Extracts email content and splits PDF attachments into individual items for processing.\n3. **Loop: Upload + OCR each PDF** — Each PDF is uploaded to Autype, OCR'd via Lens, and the extracted text is collected.\n4. **Combine Results** — All OCR texts are merged with the original request text.\n5. **Download Syntax Reference** — Fetches Autype Extended Markdown syntax so the AI knows the output format.\n6. **AI Assistant** — Processes the request (summarize, compare, draft, research), writes a document in Autype Extended Markdown.\n7. **Render PDF** — Autype renders the markdown to a branded PDF with company styling.\n8. **Send via Email** — The PDF is emailed back to the sender via SMTP.\n\n### Requirements\n- Self-hosted n8n (community nodes)\n- `n8n-nodes-autype` community node\n- Autype API key\n- OpenRouter API key (or OpenAI/Anthropic)\n- Firecrawl API key\n- SerpAPI key\n- SMTP credentials (for email delivery)\n\n> **Note:** This workflow uses the [Autype](https://www.npmjs.com/package/n8n-nodes-autype) community node. Community nodes are only available on **self-hosted n8n instances**. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for email/config section |  |  | ### 1. Email Input & Config\nThe IMAP Email Trigger monitors your inbox for incoming document requests. PDF attachments are automatically detected. The Set Company Config node defines branding variables. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for OCR loop |  |  | ### 2. PDF Processing Loop\nEach uploaded PDF is processed sequentially: uploaded to Autype via the community node, OCR'd via Autype Lens (HTTP Request), then the extracted markdown text is collected. After all PDFs are processed, the results are combined into a single context string.\n\n**Lens OCR setup:** The Lens OCR and Poll OCR Status nodes use HTTP Request with **Generic Auth Type → Header Auth**. Create a Header Auth credential with **Name: `X-API-Key`** and **Value: your Autype API key**. A dedicated Autype community node for Lens OCR is coming soon. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for AI block |  |  | ### 3. AI Document Assistant\nThe AI Assistant receives the request text, all OCR content from attached PDFs, and the Autype Extended Markdown syntax reference. It can summarize, compare, draft, or create documents from scratch — using Firecrawl and SerpAPI for research when needed. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for render/send block |  |  | ### 4. Render & Send\nThe AI output is cleaned and rendered to a branded PDF via Autype Render from Markdown. Company styling is applied via a defaults JSON (fonts, colors, heading styles, table styles, header with logo, footer with page numbers). The PDF is sent back to the sender via SMTP. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites

1. Use a **self-hosted n8n** instance.
2. Install required community/custom nodes:
   - `n8n-nodes-autype`
   - Firecrawl node package if not already available
   - LangChain/OpenRouter-capable nodes as supported by your n8n version
3. Prepare credentials:
   - **IMAP account**
   - **SMTP account**
   - **Autype account**
   - **Autype Header Auth** credential for HTTP Request nodes
   - **OpenRouter account**
   - **SerpAPI account**
   - **Firecrawl account**

## Credential Setup

### 1. IMAP
Create an IMAP credential for the mailbox that will receive requests.

### 2. SMTP
Create an SMTP credential for outbound email delivery.

### 3. Autype account
Create the Autype credential used by the Autype community nodes.

### 4. Autype Header Auth
Create an HTTP Header Auth credential with:
- **Header Name:** `X-API-Key`
- **Header Value:** your Autype API key

This credential is used by:
- `Autype Lens OCR`
- `Poll OCR Status`

### 5. OpenRouter
Create an OpenRouter API credential.

### 6. SerpAPI
Create a SerpAPI credential.

### 7. Firecrawl
Create a Firecrawl credential.

---

## Build Steps

### 1. Add the trigger node
1. Create an **Email Read IMAP** node.
2. Name it **New Email Received**.
3. Enable **Download Attachments**.
4. Select your **IMAP account** credential.

### 2. Add company configuration
5. Create a **Set** node named **Set Company Config**.
6. Connect `New Email Received -> Set Company Config`.
7. Enable keeping other fields.
8. Add these string fields:
   - `companyName`: `Your Company Name`
   - `companyLogoUrl`: `https://placehold.co/50x50/orange/white`
   - `brandColor`: `#1a5276`

### 3. Extract email content and split PDFs
9. Create a **Code** node named **Extract & Split PDFs**.
10. Connect `Set Company Config -> Extract & Split PDFs`.
11. Paste code equivalent to the following logic:
   - Read first input item
   - Extract:
     - subject
     - plain text or fallback body
     - sender email and sender name from `from`
   - Build `requestText = subject + body`
   - Scan binary attachments for PDFs by MIME type or `.pdf`
   - If no PDFs, output one item with:
     - `requestText`
     - `senderEmail`
     - `senderName`
     - `hasPdfs = false`
     - `pdfCount = 0`
   - If PDFs exist, output one item per PDF with:
     - same request metadata
     - `hasPdfs = true`
     - `pdfCount`
     - `pdfIndex`
     - `pdfFileName`
     - binary under key `data`

### 4. Add branch for PDF / no-PDF behavior
12. Create an **If** node named **Has PDFs?**.
13. Connect `Extract & Split PDFs -> Has PDFs?`.
14. Configure condition:
   - Boolean
   - left value: `{{ $json.hasPdfs }}`
   - operation: `true`

### 5. Add the no-PDF branch
15. Create a **Code** node named **Prepare Text Only**.
16. Connect the **false** output of `Has PDFs?` to it.
17. Configure it to return:
   - `requestText`
   - `senderEmail`
   - `pdfContent = ''`
   - `pdfCount = 0`
   - `hasPdfs = false`

### 6. Add the PDF loop
18. Create a **Split In Batches** node named **Loop Over PDFs**.
19. Connect the **true** output of `Has PDFs?` to it.
20. Leave reset disabled.

### 7. Upload each PDF to Autype
21. Create an **Autype** node named **Upload PDF to Autype**.
22. Connect `Loop Over PDFs -> Upload PDF to Autype`.
23. Configure:
   - Resource: `file`
24. Select the **Autype account** credential.
25. Ensure the node uses the incoming binary file (`data`).

### 8. Start OCR in Autype Lens
26. Create an **HTTP Request** node named **Autype Lens OCR**.
27. Connect `Upload PDF to Autype -> Autype Lens OCR`.
28. Configure:
   - Method: `POST`
   - URL: `https://api.autype.com/api/v1/dev/tools/lens/ocr`
   - Send body: yes
   - Body type: JSON
   - Authentication: Generic Credential Type
   - Generic Auth Type: Header Auth
29. Use the **Autype API (Header Auth)** credential.
30. JSON body:
   ```json
   {
     "fileId": "{{ $json.id }}",
     "outputFormat": "md"
   }
   ```

### 9. Wait before polling
31. Create a **Wait** node named **Wait for OCR**.
32. Connect `Autype Lens OCR -> Wait for OCR`.
33. Set the wait amount to `8` using your preferred/default time unit as supported by your n8n version.

### 10. Poll OCR status
34. Create an **HTTP Request** node named **Poll OCR Status**.
35. Connect `Wait for OCR -> Poll OCR Status`.
36. Configure:
   - Method: GET
   - URL:
     `https://api.autype.com/api/v1/dev/tools/jobs/{{ $('Autype Lens OCR').item.json.id }}`
   - Authentication: Generic Credential Type
   - Generic Auth Type: Header Auth
37. Select the same **Autype API (Header Auth)** credential.

### 11. Normalize OCR results
38. Create a **Code** node named **Extract OCR Text**.
39. Connect `Poll OCR Status -> Extract OCR Text`.
40. Configure logic to:
   - read OCR status
   - retrieve current PDF filename from `Loop Over PDFs`
   - if status is `COMPLETED`, extract markdown/text from result or metadata
   - truncate to 15000 chars
   - if `FAILED`, return fallback text
   - else return “still processing” fallback text

### 12. Close the PDF loop
41. Connect `Extract OCR Text -> Loop Over PDFs` to continue iterating.

### 13. Combine all OCR outputs
42. Create a **Code** node named **Combine All OCR Results**.
43. Connect the loop completion output of `Loop Over PDFs -> Combine All OCR Results`.
44. Configure code to:
   - collect `$('Extract OCR Text').all()`
   - collect original request data from `$('Extract & Split PDFs').first().json`
   - format one markdown section per PDF
   - return:
     - `requestText`
     - `senderEmail`
     - `pdfContent`
     - `pdfCount`
     - `hasPdfs = true`

### 14. Download markdown syntax reference
45. Create an **HTTP Request** node named **Download Markdown Syntax**.
46. Connect:
   - `Prepare Text Only -> Download Markdown Syntax`
   - `Combine All OCR Results -> Download Markdown Syntax`
47. Configure:
   - Method: GET
   - URL: `https://autype.com/llm-resources/markdown-syntax.md`
   - Response format: Text

### 15. Merge all context
48. Create a **Code** node named **Merge Context**.
49. Connect `Download Markdown Syntax -> Merge Context`.
50. Configure it to:
   - read syntax text from current input
   - try to read context from `Combine All OCR Results`
   - otherwise from `Prepare Text Only`
   - otherwise fallback to `Extract & Split PDFs`
   - add `markdownSyntax` truncated to 12000 chars

### 16. Add the language model
51. Create an **OpenRouter Chat Model** node named **OpenRouter Chat Model**.
52. Set model to `z-ai/glm-5`.
53. Assign your **OpenRouter account** credential.

### 17. Add the web search tool
54. Create a **SerpAPI Tool** node named **SerpAPI**.
55. Assign your **SerpAPI account** credential.

### 18. Add the web scraping tool
56. Create a **Firecrawl Tool** node named **/scrape in Firecrawl**.
57. Configure:
   - Operation: `scrape`
   - URL: use AI-provided URL through the node’s AI dynamic input mechanism
58. Assign your **Firecrawl account** credential.

### 19. Add the AI agent
59. Create an **AI Agent** node named **AI Document Assistant**.
60. Connect `Merge Context -> AI Document Assistant`.
61. Connect:
   - `OpenRouter Chat Model -> AI Document Assistant` via AI language model port
   - `SerpAPI -> AI Document Assistant` via AI tool port
   - `/scrape in Firecrawl -> AI Document Assistant` via AI tool port
62. Set prompt mode to defined/manual prompt.
63. Set `maxIterations` to `6`.
64. Paste a user prompt with these behaviors:
   - include request text
   - include OCR content if PDFs exist
   - instruct summarize / compare / rewrite / create-from-scratch behavior
   - require Autype Extended Markdown output only
   - forbid code fences and explanations outside the document
65. Paste a system message with these rules:
   - professional document assistant behavior
   - max 5 tool calls total
   - prefer PDF content as primary source if attachments exist
   - use SerpAPI only for research
   - use Firecrawl only for known URLs
   - include markdown syntax reference
   - require structured document output

### 20. Prepare render payload
66. Create a **Code** node named **Prepare Render Payload**.
67. Connect `AI Document Assistant -> Prepare Render Payload`.
68. Configure logic to:
   - read AI output from `output` or `text`
   - strip code fences if present
   - build a safe filename from request text
   - append current date
   - include:
     - `markdown`
     - `filename`
     - `title`
     - `senderEmail`
     - `companyName`
     - `companyLogoUrl`
     - `brandColor`

### 21. Render branded PDF
69. Create an **Autype** node named **Render Branded PDF**.
70. Connect `Prepare Render Payload -> Render Branded PDF`.
71. Configure:
   - Operation: `renderMarkdown`
   - Content: `{{ $json.markdown }}`
   - Download output: enabled
   - Size: `A4`
   - Orientation: `portrait`
   - Title: `{{ $json.title }}`
72. Set the **defaults** JSON expression to include:
   - base font settings
   - heading styles
   - table styles
   - header logo/company name
   - footer with page numbers
   - brand color references from `{{ $json.brandColor }}`

### 22. Email the final report
73. Create an **Email Send** node named **Send Report via Email**.
74. Connect `Render Branded PDF -> Send Report via Email`.
75. Configure:
   - Attachments: `data`
   - Subject:
     `Your Document: {{ $('Prepare Render Payload').item.json.title }}`
   - To:
     preferably the parsed sender email, though the source workflow uses `{{ $('New Email Received').item.json.from }}`
   - From:
     `Your Name <your-email@example.com>`
76. Assign your **SMTP account** credential.

### 23. Add optional documentation sticky notes
77. Add sticky notes summarizing:
   - overall workflow purpose
   - email/config section
   - OCR section
   - AI section
   - render/send section

---

## Recommended Improvements During Rebuild

To make the workflow more robust, consider these adjustments while reproducing it:

1. **Use parsed sender email for outbound mail**
   - Replace raw `from` with `senderEmail` to avoid invalid recipient formatting.

2. **Add OCR polling retry logic**
   - Current design waits once and polls once.
   - For large PDFs, add a retry loop until status becomes `COMPLETED` or max attempts is reached.

3. **Add explicit binary field mapping checks**
   - Confirm the Autype upload node reads binary key `data`
   - Confirm the render node outputs final PDF as binary `data`

4. **Add error branches**
   - IMAP failure notifications
   - OCR failure fallback handling
   - AI failure notifications
   - SMTP failure logging

5. **Limit oversized inputs**
   - OCR text and syntax reference are already truncated, but very large multi-PDF sets may still exceed the model context window.

6. **Review render header image access**
   - The logo URL must be public and reachable by Autype rendering.

### Sub-workflow setup

This workflow does **not** invoke any sub-workflows and does not contain any `Execute Workflow` node. It has a single entry point:
- `New Email Received`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Autype community node is required and only works on self-hosted n8n instances. | https://www.npmjs.com/package/n8n-nodes-autype |
| Workflow purpose: create branded PDF documents from email requests, optionally using attached PDF OCR and AI-generated content. | Workflow overview note |
| Required services mentioned in the workflow notes: Autype, OpenRouter or equivalent LLM provider, Firecrawl, SerpAPI, SMTP. | Project requirements |
| Lens OCR uses HTTP Request nodes with Header Auth instead of a dedicated Autype community node action. | OCR implementation note |
| Recommended Autype Header Auth credential: `X-API-Key: your Autype API key`. | Configuration note |