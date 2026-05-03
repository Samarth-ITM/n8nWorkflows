Send invoice approval requests to Slack using easybits Extractor

https://n8nworkflows.xyz/workflows/send-invoice-approval-requests-to-slack-using-easybits-extractor-14513


# Send invoice approval requests to Slack using easybits Extractor

## 1. Workflow Overview

This workflow collects an invoice file through an n8n hosted form, extracts structured invoice information using the easybits Extractor community node, normalizes and enriches the extracted data, and sends an approval request to Slack as an interactive message.

Typical use cases include:
- AP/invoice intake from internal staff
- Lightweight invoice triage before accounting validation
- Slack-based approval routing based on invoice amount

The workflow is linear and consists of four main logical blocks.

### 1.1 Input Reception
A hosted n8n form accepts a file upload representing the invoice document.

### 1.2 Invoice Data Extraction
The uploaded document is sent to the easybits Extractor node, which returns structured invoice fields such as supplier, invoice number, amount, and date.

### 1.3 Field Mapping and Approval Logic
A Set node maps the extracted fields into normalized output fields and calculates approval tiering and channel-routing metadata from the invoice amount.

### 1.4 Slack Approval Notification
An HTTP Request node posts a formatted Slack message using Slack’s `chat.postMessage` API, including approval buttons for Approve, Reject, and Flag.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block starts the workflow through a hosted n8n form. It allows a user to upload an invoice file, which becomes the source document for extraction.

**Nodes Involved:**  
- Invoice Upload Form

### Node Details

#### Invoice Upload Form
- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that creates a public/hosted form and starts the workflow on submission.

- **Configuration choices:**  
  - Form title: `Invoice Upload`
  - Description: `Upload Document`
  - One form field:
    - Type: `file`
    - Label: `image`
  - No extra form options are configured

- **Key expressions or variables used:**  
  - No expressions in the node itself
  - Produces uploaded file data for downstream consumption

- **Input and output connections:**  
  - **Input:** none, this is the trigger
  - **Output:** connects to `easybits Extractor: Extract Invoice Data`

- **Version-specific requirements:**  
  - Uses `typeVersion: 2.5`
  - Requires an n8n version supporting Form Trigger hosted forms and file upload handling in this version line

- **Edge cases or potential failure types:**  
  - User submits no file
  - Uploaded file format is unsupported by downstream extractor
  - Hosted form not reachable due to instance/network configuration
  - Large file uploads may exceed n8n or reverse-proxy upload limits
  - File field is named `image`, so downstream tools may assume that binary property name or internally map from it

- **Sub-workflow reference:**  
  - None

---

## 2.2 Invoice Data Extraction

**Overview:**  
This block sends the uploaded invoice document to the easybits Extractor node, which attempts to extract structured invoice metadata from the file.

**Nodes Involved:**  
- easybits Extractor: Extract Invoice Data

### Node Details

#### easybits Extractor: Extract Invoice Data
- **Type and technical role:**  
  `@easybits/n8n-nodes-extractor.easybitsExtractor`  
  Community node used to send the uploaded document to easybits for invoice/document parsing.

- **Configuration choices:**  
  - No explicit parameters are shown in the JSON, which implies the node likely depends on configured credentials and/or default behavior from the installed community node
  - Based on the sticky-note guidance, this node is expected to be configured with:
    - easybits Pipeline ID
    - easybits API key / credential
  - It is intended to extract at least:
    - `vendor_name`
    - `invoice_number`
    - `invoice_date`
    - `total_amount`
    - `customer_name`

- **Key expressions or variables used:**  
  - No visible expressions in the node config
  - Downstream nodes expect output in the shape:
    - `$json.data.vendor_name`
    - `$json.data.invoice_number`
    - `$json.data.invoice_date`
    - `$json.data.total_amount`
    - `$json.data.customer_name`

- **Input and output connections:**  
  - **Input:** from `Invoice Upload Form`
  - **Output:** to `Map Invoice Fields`

- **Version-specific requirements:**  
  - Uses `typeVersion: 2`
  - Requires installation of the `@easybits/n8n-nodes-extractor` community package on the n8n instance
  - The exact credential UI and required parameters depend on the installed package version

- **Edge cases or potential failure types:**  
  - Community node not installed on the target n8n instance
  - Missing or invalid easybits credentials
  - Wrong Pipeline ID or mismatched field mapping in the easybits pipeline
  - OCR/extraction failures on poor-quality scans
  - File type unsupported by easybits or malformed binary input
  - Output structure differing from expected `data.*` keys, which would break downstream expressions

- **Sub-workflow reference:**  
  - None

---

## 2.3 Field Mapping and Approval Logic

**Overview:**  
This block reshapes the easybits output into a clean invoice object and applies business logic based on the extracted amount. It also prepares metadata intended for Slack routing and approval categorization.

**Nodes Involved:**  
- Map Invoice Fields

### Node Details

#### Map Invoice Fields
- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Data transformation node used to create a new structured JSON payload from extracted values.

- **Configuration choices:**  
  The node assigns the following fields:
  - `supplier_name` = `{{$json.data.vendor_name}}`
  - `invoice_number` = `{{$json.data.invoice_number}}`
  - `invoice_date` = `{{$json.data.invoice_date}}`
  - `total_amount` = `{{$json.data.total_amount}}`
  - `currency` = `EUR`
  - `approval_tier` determined by thresholds:
    - `high_value` if amount `>= 5000`
    - `medium_value` if amount `>= 1000`
    - otherwise `low_value`
  - `approver_channel` determined by thresholds:
    - `#finance-leads` if amount `>= 5000`
    - otherwise `#invoice-review`
  - `customer_name` = `{{$json.data.customer_name}}`

- **Key expressions or variables used:**  
  - `={{ $json.data.vendor_name }}`
  - `={{ $json.data.invoice_number }}`
  - `={{ $json.data.invoice_date }}`
  - `={{ $json.data.total_amount }}`
  - `={{ $json.data.total_amount >= 5000 ? 'high_value' : ($json.data.total_amount >= 1000 ? 'medium_value' : 'low_value') }}`
  - `={{ $json.data.total_amount >= 5000 ? '#finance-leads' : '#invoice-review' }}`
  - `={{ $json.data.customer_name }}`

- **Input and output connections:**  
  - **Input:** from `easybits Extractor: Extract Invoice Data`
  - **Output:** to `Send to Slack for Approval`

- **Version-specific requirements:**  
  - Uses `typeVersion: 3.4`
  - Uses assignment-style Set node fields available in recent n8n versions

- **Edge cases or potential failure types:**  
  - If `data` is missing, all mapped expressions resolve to undefined or fail depending on execution context
  - `total_amount` may be returned as a string instead of a number
    - Numeric comparisons often coerce correctly, but locale-formatted values such as `"1,234.56"` or `"1 234,56"` may produce incorrect tiering
  - `currency` is hardcoded to `EUR`, so invoices in other currencies will be mislabeled
  - `approver_channel` is computed but not actually used by the Slack node, which currently posts to a fixed channel ID
  - Missing invoice fields lead to partially empty Slack messages

- **Sub-workflow reference:**  
  - None

---

## 2.4 Slack Approval Notification

**Overview:**  
This block sends the approval request to Slack using the Slack Web API. The message contains formatted invoice details and interactive buttons, but this workflow does not include a handler for the button actions.

**Nodes Involved:**  
- Send to Slack for Approval

### Node Details

#### Send to Slack for Approval
- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Generic HTTP client node used here to call Slack’s `chat.postMessage` endpoint directly.

- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://slack.com/api/chat.postMessage`
  - Authentication: predefined credential type
  - Credential type: `slackApi`
  - Request body format: JSON
  - `sendBody`: enabled
  - Body contains:
    - `channel`: hardcoded as `"YOUR_CHANNEL_ID"`
    - `text`: fallback plain text summary
    - `blocks`: Slack Block Kit payload with:
      - Header block
      - Section block showing supplier, invoice number, amount, date, and tier
      - Action buttons:
        - Approve
        - Reject
        - Flag

- **Key expressions or variables used:**  
  In the JSON body:
  - `{{ $json.supplier_name }}`
  - `{{ $json.currency }}`
  - `{{ $json.total_amount }}`
  - `{{ $json.invoice_number }}`
  - `{{ $json.invoice_date }}`
  - `{{ $json.approval_tier === 'high_value' ? '🔴 High (€5k+)' : $json.approval_tier === 'medium_value' ? '🟡 Medium (€1k-5k)' : '🟢 Standard (<€1k)' }}`

- **Input and output connections:**  
  - **Input:** from `Map Invoice Fields`
  - **Output:** none

- **Version-specific requirements:**  
  - Uses `typeVersion: 4.3`
  - Requires a valid Slack API credential in n8n
  - The selected Slack credential must authorize posting to the target channel

- **Edge cases or potential failure types:**  
  - Slack credential missing or invalid
  - Bot token lacks `chat:write` or channel access
  - `channel` is still placeholder text `"YOUR_CHANNEL_ID"` and must be replaced before use
  - Slack may reject malformed Block Kit payloads
  - Interactive buttons require a Slack app with Interactivity enabled and a Request URL configured, but this workflow contains no webhook/action handler
  - The workflow computes `approver_channel` but does not use it in the request; routing is therefore static unless manually changed
  - If extracted values are empty, message renders with blank fields
  - API rate limits or transient Slack service errors may interrupt posting

- **Sub-workflow reference:**  
  - None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Invoice Upload Form | n8n-nodes-base.formTrigger | Public form trigger for invoice file upload |  | easybits Extractor: Extract Invoice Data | ### 🚀 Form for Invoice Upload<br>Form accepts invoice uploads (PDF, image) |
| easybits Extractor: Extract Invoice Data | @easybits/n8n-nodes-extractor.easybitsExtractor | Extracts structured invoice data from the uploaded file | Invoice Upload Form | Map Invoice Fields | ### 🤖 Data Extraction<br>easybits Extractor pulls: supplier, invoice #, date, amount |
| Map Invoice Fields | n8n-nodes-base.set | Maps extracted fields and computes approval tier/channel metadata | easybits Extractor: Extract Invoice Data | Send to Slack for Approval | ### 📊 Field Mapping<br>Maps extracted data + determines approval tier based on amount |
| Send to Slack for Approval | n8n-nodes-base.httpRequest | Posts invoice approval message to Slack with interactive buttons | Map Invoice Fields |  | ### 💬 Slack Notification<br>Sends approval request with interactive buttons |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation for upload form |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation for extraction block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation for mapping block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas annotation for Slack block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Global workflow documentation and setup guidance |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `Invoice → Slack Approval (powered by easybits)`.

2. **Add the trigger form**
   - Add a **Form Trigger** node.
   - Set:
     - **Form Title:** `Invoice Upload`
     - **Form Description:** `Upload Document`
   - Add one field:
     - **Field Type:** `File`
     - **Field Label:** `image`
   - Leave other options at default unless you need validation or custom hosting behavior.

3. **Install the easybits community node if not already installed**
   - Ensure your n8n instance supports community nodes.
   - Install the package:
     - `@easybits/n8n-nodes-extractor`
   - Restart/reload n8n if required by your environment.

4. **Prepare the easybits extraction pipeline**
   - In easybits, create an extractor pipeline for invoices.
   - Configure these expected fields:
     - `vendor_name`
     - `invoice_number`
     - `invoice_date`
     - `total_amount`
     - `customer_name`
   - Save the pipeline and collect:
     - Pipeline ID
     - API key or credential details

5. **Add the easybits extraction node**
   - Add the node **easybits Extractor**.
   - Rename it to: `easybits Extractor: Extract Invoice Data`.
   - Connect:
     - `Invoice Upload Form` → `easybits Extractor: Extract Invoice Data`
   - Configure the node according to the installed easybits node UI:
     - Select or create easybits credentials
     - Enter the pipeline ID if required
     - Ensure the node uses the uploaded file from the form submission
   - Confirm that the node outputs extracted data under a `data` object, because the downstream mapping expects:
     - `data.vendor_name`
     - `data.invoice_number`
     - `data.invoice_date`
     - `data.total_amount`
     - `data.customer_name`

6. **Add the mapping node**
   - Add a **Set** node.
   - Rename it to: `Map Invoice Fields`.
   - Connect:
     - `easybits Extractor: Extract Invoice Data` → `Map Invoice Fields`

7. **Configure mapped output fields in the Set node**
   - Create these assignments:
     1. `supplier_name` as string  
        `{{$json.data.vendor_name}}`
     2. `invoice_number` as string  
        `{{$json.data.invoice_number}}`
     3. `invoice_date` as string  
        `{{$json.data.invoice_date}}`
     4. `total_amount` as string  
        `{{$json.data.total_amount}}`
     5. `currency` as string  
        `EUR`
     6. `approval_tier` as string  
        `{{$json.data.total_amount >= 5000 ? 'high_value' : ($json.data.total_amount >= 1000 ? 'medium_value' : 'low_value')}}`
     7. `approver_channel` as string  
        `{{$json.data.total_amount >= 5000 ? '#finance-leads' : '#invoice-review'}}`
     8. `customer_name` as string  
        `{{$json.data.customer_name}}`

8. **Add the Slack posting node**
   - Add an **HTTP Request** node.
   - Rename it to: `Send to Slack for Approval`.
   - Connect:
     - `Map Invoice Fields` → `Send to Slack for Approval`

9. **Configure Slack authentication**
   - In Slack, create or configure a Slack app.
   - Add bot scopes:
     - `chat:write`
     - `chat:write.public`
   - Install the app to the workspace.
   - Copy the bot token beginning with `xoxb-`.
   - In n8n, create a **Slack API** credential using that token.
   - Attach that credential to the HTTP Request node using predefined credential authentication.

10. **Configure the HTTP Request node**
    - Set:
      - **Method:** `POST`
      - **URL:** `https://slack.com/api/chat.postMessage`
      - **Authentication:** `Predefined Credential Type`
      - **Credential Type:** `Slack API`
      - **Send Body:** enabled
      - **Specify Body:** `JSON`

11. **Paste the Slack JSON body**
    - Configure the JSON body as equivalent to:
      - `channel`: a real Slack channel ID, replacing `YOUR_CHANNEL_ID`
      - `text`: `New invoice: {{supplier}} - {{currency}} {{amount}}`
      - `blocks`: Block Kit structure containing:
        - Header: `📄 New Invoice for Approval`
        - Section with fields for supplier, invoice number, amount, date, and tier
        - Action buttons:
          - `✅ Approve` with value `approved` and action ID `invoice_approve`
          - `❌ Reject` with value `rejected` and action ID `invoice_reject`
          - `🚩 Flag` with value `flagged` and action ID `invoice_flag`

    Use these expressions inside the body:
    - `{{$json.supplier_name}}`
    - `{{$json.invoice_number}}`
    - `{{$json.currency}}`
    - `{{$json.total_amount}}`
    - `{{$json.invoice_date}}`
    - `{{$json.approval_tier === 'high_value' ? '🔴 High (€5k+)' : $json.approval_tier === 'medium_value' ? '🟡 Medium (€1k-5k)' : '🟢 Standard (<€1k)'}}`

12. **Replace the placeholder Slack channel**
    - Change `"YOUR_CHANNEL_ID"` to the actual Slack channel ID.
    - Important: the current workflow does **not** use the computed `approver_channel` field dynamically.
    - If you want dynamic routing, modify the `channel` field to reference an expression or add logic to map Slack channel IDs explicitly.

13. **Set up Slack interactivity if buttons should do something**
    - In your Slack app settings, enable **Interactivity**.
    - Provide a **Request URL** that points to a webhook or workflow capable of receiving Slack action payloads.
    - Note: this workflow only sends the message; it does **not** contain an action-processing endpoint.

14. **Optionally improve numeric reliability**
    - If easybits returns `total_amount` as a formatted string, add a cleanup/conversion step before tier calculation.
    - This avoids incorrect comparisons caused by locale-specific number formatting.

15. **Test the workflow**
    - Execute the workflow manually or activate it.
    - Open the generated form URL.
    - Upload a sample invoice file.
    - Verify:
      - easybits extracts the expected fields
      - the Set node outputs all mapped fields
      - Slack receives the message with buttons

16. **Activate the workflow**
    - Once validated, activate the workflow so the hosted form is live for real submissions.

### Expected input/output behavior

#### Input
- One uploaded invoice file from the form field labeled `image`

#### Intermediate output expected from easybits
A JSON object with a `data` structure similar to:
- `data.vendor_name`
- `data.invoice_number`
- `data.invoice_date`
- `data.total_amount`
- `data.customer_name`

#### Final output
- A Slack message posted to the configured channel with invoice details and action buttons

### Credential requirements
- **easybits credentials**
  - API key and pipeline configuration, as required by the community node
- **Slack API credentials**
  - Bot token with `chat:write`
  - Often also `chat:write.public` if posting to channels the bot is not explicitly invited to

### Constraints and implementation notes
- `currency` is hardcoded to `EUR`
- Channel routing is currently static despite the presence of `approver_channel`
- Approval buttons are UI-only unless another workflow/webhook handles Slack interactive callbacks
- The workflow has a single entry point: the Form Trigger
- There are no sub-workflows in this design

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Invoice Approval Workflow (powered by easybits + Slack) | General workflow branding |
| Upload an invoice (PDF, PNG, or JPEG) via a hosted web form. The file is sent to easybits Extractor, which extracts key invoice data. Based on the amount, an approval tier is assigned, then the invoice is posted to Slack with interactive buttons. | Functional summary |
| easybits Extractor pipeline should include fields: `vendor_name`, `invoice_number`, `invoice_date`, `total_amount`, `customer_name`. | easybits setup |
| easybits site referenced for pipeline creation: `extractor.easybits.tech` | easybits platform |
| Slack app creation link: `https://api.slack.com/apps` | Slack app setup |
| Suggested Slack scopes: `chat:write`, `chat:write.public` | Slack permissions |
| Enable Slack Interactivity and set the Request URL to your approval handler webhook. | Slack button handling |
| Approval tiers used in the workflow: Standard `< €1,000`, Medium `€1,000 – €5,000`, High `> €5,000`. | Business rules |