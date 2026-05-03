Classify Gmail invoice attachments with Google Gemini and forward to freee

https://n8nworkflows.xyz/workflows/classify-gmail-invoice-attachments-with-google-gemini-and-forward-to-freee-13237


# Classify Gmail invoice attachments with Google Gemini and forward to freee

## 1. Workflow Overview

**Purpose:** Poll Gmail for new emails, AI-classify the first PDF attachment as **invoice/receipt/other** using **Google Gemini**, and if it’s an invoice or receipt, **forward the original attachment** to **freee Accounting File Box** via email.

**Target use cases:** Automating accounting intake—routing invoices/receipts from inbound email into freee without manual triage.

### 1.1 Input Reception (Gmail polling)
- Polls Gmail every minute, downloading attachments.

### 1.2 Attachment Presence Filtering
- Quickly filters out emails that are not multipart (i.e., likely no attachments) using the `Content-Type` header.

### 1.3 PDF Text Extraction + AI Classification
- Extracts text from the first attachment (`attachment_0`) as PDF text.
- Sends extracted text + email subject/from/body to Gemini with strict JSON-only output requirements.

### 1.4 Forwarding to freee (only if target)
- Re-fetches the original email (to obtain original attachment binaries reliably).
- Forwards to freee’s File Box receiving address from env var `FREEE_FORWARDING_EMAIL`.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Gmail polling)
**Overview:** Watches Gmail for new emails every minute and includes attachments in the trigger output.

**Nodes involved:**
- **Gmail Trigger**

#### Node: Gmail Trigger
- **Type / role:** `gmailTrigger` — scheduled polling trigger for new Gmail messages.
- **Key configuration:**
  - Polling: **every minute**
  - `includeSpamTrash: false`
  - `downloadAttachments: true` (attachments are pulled into binary fields like `attachment_0`)
  - `simple: false` (expects richer Gmail payload including headers)
- **Inputs/outputs:**
  - **Output →** `Has Attachment`
- **Edge cases / failures:**
  - OAuth token expired or missing scopes (needs Gmail read permissions).
  - Large attachments may not be included or may hit size/time constraints.
  - Gmail trigger payload can vary; relying on specific headers requires `simple: false`.

---

### Block 2 — Attachment Presence Filtering
**Overview:** Filters out emails that do not appear to contain attachments, based on the `Content-Type` header.

**Nodes involved:**
- **Has Attachment**
- **No Attachment - End**

#### Node: Has Attachment
- **Type / role:** `if` — conditional routing.
- **Key configuration:**
  - Condition: checks whether `{{$json.headers?.['content-type']}}` **contains** `"multipart/mixed"`
- **Connections:**
  - **True →** `Extract Text from PDF`
  - **False →** `No Attachment - End`
- **Edge cases / failures:**
  - Some emails with attachments may use other multipart types (e.g., `multipart/related`), or header casing/structure may differ.
  - If `headers` is absent, expression resolves to `undefined`; condition safely evaluates to false, potentially skipping valid messages.

#### Node: No Attachment - End
- **Type / role:** `noOp` — terminates the non-attachment path.
- **Connections:** none
- **Edge cases / failures:** none (used as a sink node).

---

### Block 3 — PDF Text Extraction + AI Classification
**Overview:** Extracts text from the first attachment as a PDF, then asks Gemini to classify it with strict JSON output; workflow proceeds only when classification marks it as invoice/receipt.

**Nodes involved:**
- **Extract Text from PDF**
- **Classify Invoice/Receipt (AI)**
- **Is Invoice or Receipt**
- **Not Target - End**

#### Node: Extract Text from PDF
- **Type / role:** `extractFromFile` — parses a binary file and extracts text.
- **Key configuration:**
  - Operation: `pdf`
  - Binary property: `attachment_0`
- **Connections:**
  - **Input:** from `Has Attachment` (true path)
  - **Output →** `Classify Invoice/Receipt (AI)`
- **Edge cases / failures:**
  - If the first attachment is not a PDF (or is an image-only scanned PDF), extraction may fail or return little text.
  - If Gmail provides multiple attachments, only `attachment_0` is used; relevant invoice may be in `attachment_1+`.

#### Node: Classify Invoice/Receipt (AI)
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — LLM call to Gemini via LangChain integration.
- **Version:** `typeVersion 1.1` (Gemini LangChain node; ensure the same node package/version exists in your n8n).
- **Key configuration:**
  - Model: `models/gemini-2.5-flash`
  - Prompting strategy:
    - A **system/model instruction** enforcing:
      - JSON-only output
      - Fixed keys: `is_invoice_or_receipt`, `document_type`, `confidence`, `reason`
      - Single-sentence reason ≤ 80 chars
    - A user message that includes:
      - Email subject/from/body from **Gmail Trigger**
      - Attachment text from `Extract Text from PDF` output: `{{ $json.text }}`
- **Connections:**
  - **Output →** `Is Invoice or Receipt`
- **Edge cases / failures:**
  - Gemini may return non-JSON despite instruction (causing downstream `JSON.parse` to fail).
  - Rate limits / quota errors / invalid API key.
  - Very long extracted text could exceed model limits; consider truncation if needed.

#### Node: Is Invoice or Receipt
- **Type / role:** `if` — gates forwarding based on AI output.
- **Key configuration:**
  - Condition checks boolean true:
    - `={{ JSON.parse($json.content.parts[0].text).is_invoice_or_receipt }}`
- **Connections:**
  - **True →** `Get Attachment`
  - **False →** `Not Target - End`
- **Edge cases / failures:**
  - If Gemini output is not valid JSON: `JSON.parse(...)` throws an expression error and the workflow run fails.
  - If the Gemini node output structure changes (e.g., not `content.parts[0].text`), parsing will fail.

#### Node: Not Target - End
- **Type / role:** `noOp` — terminates when doc is not invoice/receipt.
- **Connections:** none

---

### Block 4 — Forwarding to freee
**Overview:** Re-downloads the email (to ensure attachment binaries are present and fresh), then forwards it to freee’s File Box receiving email with a formatted forward body and original attachment.

**Nodes involved:**
- **Get Attachment**
- **Forward to freee**

#### Node: Get Attachment
- **Type / role:** `gmail` — fetch a specific message by ID.
- **Key configuration:**
  - Operation: `get`
  - Message ID: `={{ $('Gmail Trigger').item.json.id }}`
  - `downloadAttachments: true`
  - `simple: false`
- **Connections:**
  - **Output →** `Forward to freee`
- **Why re-fetch?**
  - Trigger attachment payloads can be inconsistent or incomplete depending on Gmail API behavior; refetch ensures attachments are present for forwarding.
- **Edge cases / failures:**
  - Message ID not found (deleted/moved) between trigger and fetch.
  - OAuth scope missing for message read.
  - Large attachments might not download or may exceed limits.

#### Node: Forward to freee
- **Type / role:** `gmail` — sends an email with attachment.
- **Key configuration:**
  - To: `={{ $env.FREEE_FORWARDING_EMAIL }}`
  - Subject: `=Fwd: {{ $('Gmail Trigger').item.json.subject }}`
  - Body message includes AI classification and original email info; key expression:
    - `{{ JSON.parse($('Classify Invoice/Receipt (AI)').item.json.content.parts[0].text).document_type === 'invoice' ? 'Invoice' : 'Receipt' }}`
  - Attachments: uses binary property **`attachment_0`** from the current item (coming from `Get Attachment`)
- **Connections:** none
- **Version:** `typeVersion 2.1`
- **Edge cases / failures:**
  - If `FREEE_FORWARDING_EMAIL` env var is missing/blank, send fails.
  - If AI returns `document_type: "other"` but `is_invoice_or_receipt` was true (inconsistent), message text will label it as “Receipt” due to ternary logic.
  - If the invoice is not `attachment_0`, wrong file may be forwarded.
  - Gmail send permissions required (OAuth scope for sending).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / setup context | — | — | ## Classify and forward Gmail invoice attachments to freee with Google Gemini… (includes setup steps, env var `FREEE_FORWARDING_EMAIL`) |
| Filter Section | Sticky Note | Documents “Receive & Filter” block | — | — | ### Receive & Filter Polls Gmail every minute and filters out emails without attachments using the Content-Type header. |
| AI Section | Sticky Note | Documents “AI Classification” block | — | — | ### AI Classification Extracts text from the PDF attachment… classify as invoice, receipt, or other. |
| Forward Section | Sticky Note | Documents “Forward to freee” block | — | — | ### Forward to freee Re-fetches the original attachment and forwards the email… |
| Gmail Trigger | gmailTrigger | Poll Gmail for new emails + download attachments | — | Has Attachment | ### Receive & Filter Polls Gmail every minute and filters out emails without attachments using the Content-Type header. |
| Has Attachment | if | Filter for attachment presence via Content-Type | Gmail Trigger | Extract Text from PDF; No Attachment - End | ### Receive & Filter Polls Gmail every minute and filters out emails without attachments using the Content-Type header. |
| No Attachment - End | noOp | Stop path when no attachment | Has Attachment (false) | — | ### Receive & Filter Polls Gmail every minute and filters out emails without attachments using the Content-Type header. |
| Extract Text from PDF | extractFromFile | Extract text from first PDF attachment | Has Attachment (true) | Classify Invoice/Receipt (AI) | ### AI Classification Extracts text from the PDF attachment… classify as invoice, receipt, or other. |
| Classify Invoice/Receipt (AI) | Google Gemini (LangChain) | LLM classification, JSON-only output | Extract Text from PDF | Is Invoice or Receipt | ### AI Classification Extracts text from the PDF attachment… classify as invoice, receipt, or other. |
| Is Invoice or Receipt | if | Gate forwarding based on AI JSON field | Classify Invoice/Receipt (AI) | Get Attachment; Not Target - End | ### AI Classification Extracts text from the PDF attachment… classify as invoice, receipt, or other. |
| Not Target - End | noOp | Stop path when not invoice/receipt | Is Invoice or Receipt (false) | — | ### AI Classification Extracts text from the PDF attachment… classify as invoice, receipt, or other. |
| Get Attachment | gmail | Fetch message again with attachments | Is Invoice or Receipt (true) | Forward to freee | ### Forward to freee Re-fetches the original attachment and forwards the email… |
| Forward to freee | gmail | Send email to freee with attachment | Get Attachment | — | ### Forward to freee Re-fetches the original attachment and forwards the email… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create environment variable**
   - On the n8n host, set: `FREEE_FORWARDING_EMAIL=<your freee File Box receiving email>`
   - In freee: *freee Accounting → File Box → Settings* to find the receiving address.

2. **Create “Gmail Trigger” node**
   - Node type: **Gmail Trigger**
   - Configure:
     - Poll interval: **Every minute**
     - Download attachments: **enabled**
     - Include spam/trash: **disabled**
     - Simple mode: **disabled** (to get headers)
   - Set **Gmail OAuth2 credentials** with scopes to read mail.

3. **Create “Has Attachment” IF node**
   - Node type: **IF**
   - Condition (String → contains):
     - Left: `={{ $json.headers?.['content-type'] }}`
     - Right: `multipart/mixed`
   - Connect: **Gmail Trigger → Has Attachment**

4. **Create “No Attachment - End” node**
   - Node type: **No Operation (NoOp)**
   - Connect: **Has Attachment (false) → No Attachment - End**

5. **Create “Extract Text from PDF” node**
   - Node type: **Extract From File**
   - Operation: **PDF**
   - Binary property name: `attachment_0`
   - Connect: **Has Attachment (true) → Extract Text from PDF**

6. **Create “Classify Invoice/Receipt (AI)” node**
   - Node type: **Google Gemini (LangChain)** (the `@n8n/n8n-nodes-langchain.googleGemini` node)
   - Model: `models/gemini-2.5-flash`
   - Messages:
     - **Model/System** message: use the strict JSON-only classifier instruction (as in the workflow).
     - **User** message: include subject/from/body from *Gmail Trigger* and attachment text via `{{ $json.text }}`
   - Configure **Gemini API credentials** (API key or configured credential method supported by your n8n).
   - Connect: **Extract Text from PDF → Classify Invoice/Receipt (AI)**

7. **Create “Is Invoice or Receipt” IF node**
   - Node type: **IF**
   - Condition: Boolean → is true
     - Left value:
       - `={{ JSON.parse($json.content.parts[0].text).is_invoice_or_receipt }}`
   - Connect: **Classify Invoice/Receipt (AI) → Is Invoice or Receipt**

8. **Create “Not Target - End” node**
   - Node type: **NoOp**
   - Connect: **Is Invoice or Receipt (false) → Not Target - End**

9. **Create “Get Attachment” node**
   - Node type: **Gmail**
   - Operation: **Get**
   - Message ID:
     - `={{ $('Gmail Trigger').item.json.id }}`
   - Options:
     - Download attachments: **enabled**
     - Simple mode: **disabled**
   - Use the **same Gmail OAuth2 credentials** (read access).
   - Connect: **Is Invoice or Receipt (true) → Get Attachment**

10. **Create “Forward to freee” node**
   - Node type: **Gmail**
   - Operation: **Send**
   - To:
     - `={{ $env.FREEE_FORWARDING_EMAIL }}`
   - Subject:
     - `=Fwd: {{ $('Gmail Trigger').item.json.subject }}`
   - Message body: include original email fields and AI classification; ensure it references:
     - `$('Classify Invoice/Receipt (AI)')...content.parts[0].text` for the JSON
     - `$('Gmail Trigger')...` for from/date/subject/text
   - Attachments:
     - Attach binary property `attachment_0` (coming from **Get Attachment** output)
   - Use **Gmail OAuth2 credentials** with send permission.
   - Connect: **Get Attachment → Forward to freee**

11. **Add sticky notes (optional but recommended)**
   - Add notes for: overview/setup, receive/filter, AI classification, forwarding.

12. **Activate workflow**
   - Ensure Gmail + Gemini credentials are valid.
   - Send a test email with a PDF invoice attached and confirm forwarding.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically analyze email attachments with AI and forward invoices/receipts to freee's File Box. | Workflow intent (from “Workflow Overview” note) |
| Setup steps: get freee receiving email → set `FREEE_FORWARDING_EMAIL` → configure Gmail OAuth2 + Google Gemini API → activate. | Operational setup (from “Workflow Overview” note) |
| Filtering is based on `Content-Type` containing `multipart/mixed`. | May need adjustment depending on email formats |
| The workflow assumes the target document is `attachment_0` and is a PDF. | Consider extending for multiple attachments / non-PDF receipts |