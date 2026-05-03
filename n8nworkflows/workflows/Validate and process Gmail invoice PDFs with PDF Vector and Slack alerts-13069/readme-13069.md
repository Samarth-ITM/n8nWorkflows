Validate and process Gmail invoice PDFs with PDF Vector and Slack alerts

https://n8nworkflows.xyz/workflows/validate-and-process-gmail-invoice-pdfs-with-pdf-vector-and-slack-alerts-13069


# Validate and process Gmail invoice PDFs with PDF Vector and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Validate and process Gmail invoice PDFs with PDF Vector and Slack alerts  
**Workflow name (in JSON):** Invoice Processing with Validation

**Purpose:**  
Automatically monitors Gmail for new emails, downloads PDF attachments, extracts invoice data using **PDF Vector**, validates invoice math and required fields, then routes results:
- **Valid invoices** → appended to a Google Sheet tab + Slack confirmation
- **Invalid/needs review** → appended to a flagged tab + Slack alert with errors

### 1.1 Logical Blocks
1. **Input Reception & File Acquisition**  
   Watches Gmail and downloads attachments.
2. **Binary Normalization for Extraction**  
   Renames the downloaded binary field to match what PDF Vector expects.
3. **AI Extraction & Deterministic Validation**  
   Extracts structured invoice data via PDF Vector, then validates calculations and required fields via code.
4. **Routing, Logging, and Notifications**  
   IF node splits valid vs flagged, logs to Google Sheets, notifies via Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & File Acquisition
**Overview:**  
Triggers on incoming Gmail messages and fetches the full message while downloading attachments into binary properties.

**Nodes involved:**  
- Sticky Note1 (documentation)  
- Gmail Trigger  
- Get attachment

#### Node: Sticky Note1
- **Type/role:** Sticky Note (documentation)
- **Content:** “## 1. Get invoice — Downloads PDF attachment from incoming emails”
- **Connections:** None (visual only)
- **Edge cases:** None (non-executing)

#### Node: Gmail Trigger
- **Type/role:** `gmailTrigger` — entry point; polls Gmail for new messages.
- **Configuration (interpreted):**
  - Polling frequency: **every minute**
  - Include spam/trash: **false**
  - No query/label filter is configured in the provided JSON (so it may trigger for many emails, not only invoices).
- **Credentials:** Gmail OAuth2 (`YOUR_GMAIL_CREDENTIAL_ID`)
- **Outputs:**  
  - Main → **Get attachment**
- **Edge cases / failures:**
  - OAuth token expiration / insufficient Gmail scopes
  - High email volume causing many executions
  - If you expect only PDF invoice emails, lack of a Gmail search filter is a common cause of noise.

#### Node: Get attachment
- **Type/role:** `gmail` — retrieves message details and downloads attachments to binary.
- **Configuration (interpreted):**
  - Operation: **Get message**
  - Message ID: `={{ $json.id }}`
  - Download attachments: **enabled**
  - Attachment binary prefix: `attachment_` (so first file usually becomes `binary.attachment_0`)
  - `simple: false` (returns more complete message structure)
- **Credentials:** Gmail OAuth2 (`YOUR_GMAIL_CREDENTIAL_ID`)
- **Inputs:** Gmail Trigger
- **Outputs:** Rename binary
- **Edge cases / failures:**
  - Email has **no attachments** → downstream rename/extract may fail
  - Multiple attachments → only the first is handled later (see rename step)
  - Non-PDF attachments → PDF Vector may fail or extract garbage
  - Gmail API limits / transient errors

---

### Block 2 — Binary Normalization for Extraction
**Overview:**  
Renames the first downloaded attachment binary key from `attachment_0` to `data`, aligning with PDF Vector node configuration (`binaryPropertyName: data`).

**Nodes involved:**  
- Sticky Note2 (documentation)  
- Rename binary

#### Node: Sticky Note2
- **Type/role:** Sticky Note (documentation)
- **Content:** “## 2. Extract & validate — AI reads invoice, then checks all calculations”
- **Connections:** None

#### Node: Rename binary
- **Type/role:** `code` — transforms incoming items’ binary structure.
- **Configuration (interpreted):**
  - Mode: **Run Once for All Items**
  - Logic:
    - If `item.binary.attachment_0` exists:
      - Copies it to `item.binary.data`
      - Deletes `item.binary.attachment_0`
- **Inputs:** Get attachment
- **Outputs:** PDF Vector - Extract invoice
- **Key variables/fields:**
  - Uses `$input.all()`; modifies `item.binary`
- **Edge cases / failures:**
  - If the attachment is not in `attachment_0` (e.g., multiple files, different indexing, or no attachments), `binary.data` will not exist → PDF Vector node will fail due to missing binary property.
  - If the PDF is not the first attachment, the workflow will process the wrong file.

---

### Block 3 — AI Extraction & Deterministic Validation
**Overview:**  
PDF Vector extracts a structured invoice object. A Code node then validates line-item arithmetic and total/subtotal relationships and produces a normalized result payload.

**Nodes involved:**  
- PDF Vector - Extract invoice  
- Validate invoice

#### Node: PDF Vector - Extract invoice
- **Type/role:** `pdfVector` (community/custom node `n8n-nodes-pdfvector.pdfVector`) — document extraction with schema-constrained output.
- **Configuration (interpreted):**
  - Resource: **document**
  - Operation: **extract**
  - Input type: **file**
  - Binary property name: **data**
  - Prompt: Extract invoice number/date/vendor/line items/subtotal/tax/total/bank details.
  - Schema: JSON schema enforcing:
    - Required: `invoice_number`, `vendor_name`, `total_amount`
    - `line_items` array objects with `description`, `quantity`, `unit_price`, `amount`
    - `additionalProperties: false` (rejects extra keys in structured output)
- **Credentials:** PDF Vector API key (`YOUR_PDFVECTOR_CREDENTIAL_ID`)
- **Inputs:** Rename binary
- **Outputs:** Validate invoice
- **Version-specific notes:**
  - Requires the **PDF Vector node package** installed in n8n and compatible with your n8n version.
- **Edge cases / failures:**
  - Missing `binary.data` → node error
  - Non-PDF or corrupted PDF → extraction failure
  - Schema strictness (`additionalProperties: false`) can cause parsing failures if the model returns unexpected keys
  - Vendor invoices with unusual layouts may produce incomplete line items or wrong amounts

#### Node: Validate invoice
- **Type/role:** `code` — deterministic validation and normalization; produces a single summary item.
- **Configuration (interpreted):**
  - Mode: **Run Once for All Items**
  - Reads extracted invoice at: `const invoice = $input.first().json.data;`
    - Assumes the PDF Vector output has a `json.data` object containing the invoice fields.
  - Validation performed:
    1. **Line item math:** checks `quantity * unit_price ≈ amount` with tolerance **0.01**
    2. **Subtotal check:** sum(line item amounts) ≈ stated subtotal (if provided)
    3. **Total check:** (subtotal used for calc + tax) ≈ total
    4. **Required fields:** invoice_number, vendor_name, total_amount
  - Produces:
    - `validation_status`: `"valid"` or `"needs_review"`
    - `is_valid`: boolean
    - `validation_errors`: array of `{type, message}`
    - `error_count`
    - `invoice_data`: the extracted invoice object
    - `items_summary`: human-readable list of items
    - `processed_at`: ISO timestamp
    - `email_subject`: from `$('Gmail Trigger').first().json.subject || 'Unknown'`
- **Inputs:** PDF Vector - Extract invoice
- **Outputs:** Valid?
- **Key expressions/variables:**
  - Tolerance hardcoded: `0.01`
  - Cross-node reference: `$('Gmail Trigger').first()...`
- **Edge cases / failures:**
  - If PDF Vector output structure differs (no `json.data`) → runtime error
  - If numeric fields arrive as strings (e.g., `"12.34"`) comparisons may behave unexpectedly; consider coercion
  - If `line_items` missing/empty → forced error `missing_line_items`
  - Floating point rounding: tolerance helps, but large invoices may accumulate rounding differences

---

### Block 4 — Routing, Logging, and Notifications
**Overview:**  
Routes based on `is_valid`. Appends to the appropriate Google Sheets tab and posts a Slack message to the configured channel.

**Nodes involved:**  
- Sticky Note3 (documentation)  
- Valid?  
- Log valid  
- Slack success  
- Log flagged  
- Slack alert

#### Node: Sticky Note3
- **Type/role:** Sticky Note (documentation)
- **Content:** “## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert.”
- **Connections:** None

#### Node: Valid?
- **Type/role:** `if` — branching control node.
- **Configuration (interpreted):**
  - Condition: `={{ $json.is_valid }}` **equals** `true` (boolean strict)
  - True branch (output 0) → Log valid
  - False branch (output 1) → Log flagged
- **Inputs:** Validate invoice
- **Outputs:** Log valid (true), Log flagged (false)
- **Edge cases / failures:**
  - If `is_valid` is missing or not boolean (e.g., `"true"` as string), strict validation can route incorrectly.

#### Node: Log valid
- **Type/role:** `googleSheets` — appends a row into the **Invoices** tab.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet/tab: **Invoices**
  - Column mapping (selected highlights):
    - Invoice Number ← `invoice_data.invoice_number`
    - Vendor ← `invoice_data.vendor_name`
    - Date ← `invoice_data.invoice_date`
    - Due Date ← `invoice_data.due_date || 'N/A'`
    - Subtotal ← `invoice_data.subtotal || 0`
    - Tax ← `invoice_data.tax_amount || 0`
    - Total ← `invoice_data.total_amount`
    - Items ← `items_summary`
    - Status ← `"Valid"`
    - Processed ← `processed_at.split('T')[0]` (YYYY-MM-DD)
- **Credentials:** Google Sheets OAuth2 (`YOUR_SHEETS_CREDENTIAL_ID`)
- **Inputs:** Valid? (true branch)
- **Outputs:** Slack success
- **Edge cases / failures:**
  - Sheet/tab name mismatch (“Invoices” must exist)
  - Column headers must match mapping expectations in the node configuration
  - OAuth/scopes issues
  - API quota limits

#### Node: Slack success
- **Type/role:** `slack` — posts a confirmation message.
- **Configuration (interpreted):**
  - Auth: OAuth2
  - Channel: `YOUR_SLACK_CHANNEL_ID`
  - Message includes vendor, invoice number, total.
  - Uses expressions referencing the IF node item:
    - `{{ $('Valid?').item.json.invoice_data.vendor_name }}`
- **Credentials:** Slack OAuth2 (`YOUR_SLACK_CREDENTIAL_ID`)
- **Inputs:** Log valid
- **Outputs:** none
- **Edge cases / failures:**
  - Channel ID invalid or bot not in channel
  - Slack permission scopes missing (e.g., `chat:write`)
  - Expression references can break if node name changes (“Valid?”)

#### Node: Log flagged
- **Type/role:** `googleSheets` — appends a row into the **Flagged Invoices** tab.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet/tab: **Flagged Invoices**
  - Column mapping:
    - Invoice Number, Vendor, Total
    - Error Count ← `error_count`
    - Errors ← `validation_errors.map(e => e.message).join('; ')`
    - Email Subject ← `email_subject`
    - Processed date ← `processed_at.split('T')[0]`
- **Credentials:** Google Sheets OAuth2 (`YOUR_SHEETS_CREDENTIAL_ID`)
- **Inputs:** Valid? (false branch)
- **Outputs:** Slack alert
- **Edge cases / failures:**
  - Tab missing or columns mismatch
  - Very long “Errors” field may exceed cell limits in extreme cases

#### Node: Slack alert
- **Type/role:** `slack` — posts an alert message for review.
- **Configuration (interpreted):**
  - Auth: OAuth2
  - Channel: `YOUR_SLACK_CHANNEL_ID`
  - Message includes vendor, invoice number, and errors joined by commas.
  - Expressions reference the IF node item (same pattern as success).
- **Credentials:** Slack OAuth2 (`YOUR_SLACK_CREDENTIAL_ID`)
- **Inputs:** Log flagged
- **Outputs:** none
- **Edge cases / failures:**
  - Same Slack permission/channel considerations as success message
  - Error message list may become lengthy and truncated in Slack

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | High-level workflow notes & setup instructions |  |  | 🧾 Invoice Validation from Gmail — How it works (Gmail trigger → PDF Vector → validation → Sheets + Slack). Setup steps: Gmail OAuth2, PDF Vector key (pdfvector.com/api-keys), Google Sheet tabs “Invoices”/“Flagged Invoices”, Slack channels. Customization: ±$0.01 tolerance, required fields, swap Slack. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section header (Get invoice) |  |  | ## 1. Get invoice — Downloads PDF attachment from incoming emails |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section header (Extract & validate) |  |  | ## 2. Extract & validate — AI reads invoice, then checks all calculations |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section header (Route & notify) |  |  | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for new emails |  | Get attachment | ## 1. Get invoice — Downloads PDF attachment from incoming emails |
| Get attachment | n8n-nodes-base.gmail | Fetch message + download attachments | Gmail Trigger | Rename binary | ## 1. Get invoice — Downloads PDF attachment from incoming emails |
| Rename binary | n8n-nodes-base.code | Rename `attachment_0` → `data` for extractor | Get attachment | PDF Vector - Extract invoice | ## 2. Extract & validate — AI reads invoice, then checks all calculations |
| PDF Vector - Extract invoice | n8n-nodes-pdfvector.pdfVector | Extract structured invoice data from PDF | Rename binary | Validate invoice | ## 2. Extract & validate — AI reads invoice, then checks all calculations |
| Validate invoice | n8n-nodes-base.code | Validate totals/subtotals/line math + required fields | PDF Vector - Extract invoice | Valid? | ## 2. Extract & validate — AI reads invoice, then checks all calculations |
| Valid? | n8n-nodes-base.if | Route valid vs needs_review | Validate invoice | Log valid; Log flagged | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |
| Log valid | n8n-nodes-base.googleSheets | Append valid invoice row to “Invoices” | Valid? (true) | Slack success | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |
| Slack success | n8n-nodes-base.slack | Slack confirmation for valid invoices | Log valid |  | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |
| Log flagged | n8n-nodes-base.googleSheets | Append flagged invoice row to “Flagged Invoices” | Valid? (false) | Slack alert | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |
| Slack alert | n8n-nodes-base.slack | Slack alert with validation errors | Log flagged |  | ## 3. Route & notify — Valid → log + confirm. Invalid → flag + alert. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Invoice Processing with Validation** (or your preferred name).

2. **Add a Gmail Trigger node**
   - Node type: **Gmail Trigger**
   - Configure polling: **Every minute**
   - Filters: keep **Include Spam/Trash = false**
   - **Credentials:** Add Gmail OAuth2 credentials and authorize the Gmail account.
   - (Recommended customization) Add a Gmail query/label filter to target invoice emails and/or PDFs.

3. **Add a Gmail “Get message” node**
   - Node type: **Gmail**
   - Operation: **Get**
   - Message ID: set expression to the trigger’s message id: `={{ $json.id }}`
   - Options:
     - Enable **Download Attachments**
     - Set attachment prefix to `attachment_` (default used here)
   - Use the same **Gmail OAuth2** credentials.
   - Connect: **Gmail Trigger → Get attachment**

4. **Add a Code node to rename the binary property**
   - Node type: **Code**
   - Mode: **Run Once for All Items**
   - Paste logic to rename `binary.attachment_0` to `binary.data` (so the extractor can read `data`).
   - Connect: **Get attachment → Rename binary**

5. **Install/configure PDF Vector node**
   - Ensure the **PDF Vector n8n node** (`n8n-nodes-pdfvector`) is installed in your n8n environment.
   - Add node: **PDF Vector**
   - Resource: **document**
   - Operation: **extract**
   - Input type: **file**
   - Binary property name: **data**
   - Prompt: instruct to extract invoice fields (number, dates, vendor, line items, subtotal, tax, total, bank details).
   - Schema: define a JSON schema with required fields (at least invoice_number, vendor_name, total_amount) and line_items structure.
   - **Credentials:** create PDF Vector API credentials using an API key from: https://pdfvector.com/api-keys
   - Connect: **Rename binary → PDF Vector - Extract invoice**

6. **Add a Code node for validation**
   - Node type: **Code**
   - Mode: **Run Once for All Items**
   - Implement checks:
     - For each line item: `quantity * unit_price` matches `amount` within tolerance (±0.01)
     - Sum of line item amounts matches subtotal (if subtotal present)
     - Subtotal + tax matches total
     - Required fields exist
   - Output a single JSON item containing:
     - `is_valid`, `validation_errors`, `error_count`, `invoice_data`, `items_summary`, `processed_at`, `email_subject`
   - Connect: **PDF Vector - Extract invoice → Validate invoice**

7. **Add an IF node for routing**
   - Node type: **IF**
   - Condition: Boolean equals
     - Left: `={{ $json.is_valid }}`
     - Right: `true`
   - Connect: **Validate invoice → Valid?**

8. **Prepare Google Sheets**
   - Create a spreadsheet with two tabs:
     - **Invoices**
     - **Flagged Invoices**
   - Ensure the column headers match what you plan to append (e.g., Invoice Number, Vendor, Total, etc.).

9. **Add Google Sheets node for valid invoices**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Document: your spreadsheet ID
   - Sheet: **Invoices**
   - Map columns using expressions from `invoice_data`, plus status and processed date.
   - **Credentials:** Google Sheets OAuth2 credentials authorized for that spreadsheet.
   - Connect: **Valid? (true output) → Log valid**

10. **Add Slack node for success**
   - Node type: **Slack**
   - Operation: send message to channel
   - Channel: select your target channel (store as channel ID)
   - Text: include vendor, invoice number, total (from the current item)
   - **Credentials:** Slack OAuth2 (ensure bot has `chat:write` and is in channel)
   - Connect: **Log valid → Slack success**

11. **Add Google Sheets node for flagged invoices**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Sheet: **Flagged Invoices**
   - Columns: vendor, invoice number, total, error_count, joined error messages, email_subject, processed date
   - Connect: **Valid? (false output) → Log flagged**

12. **Add Slack node for alerts**
   - Node type: **Slack**
   - Channel: same or dedicated alert channel
   - Text: vendor, invoice number, and joined validation error messages
   - Connect: **Log flagged → Slack alert**

13. **(Optional) Add sticky notes**
   - Add sticky notes for the three sections and an overall note to document setup and customization.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PDF Vector API keys | https://pdfvector.com/api-keys |
| Google Sheet must contain two tabs: “Invoices” and “Flagged Invoices” | Used by the two Google Sheets append nodes |
| Validation tolerance is ±$0.01 | Implemented in the “Validate invoice” Code node |
| Slack can be swapped for email/Teams | Mentioned in the main sticky note customization section |
| Current workflow assumes the invoice PDF is the first attachment (`attachment_0`) | If multiple attachments exist, update the rename logic to select the correct PDF |

