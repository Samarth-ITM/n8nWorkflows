Log invoice approval decisions from Slack to Google Sheets

https://n8nworkflows.xyz/workflows/log-invoice-approval-decisions-from-slack-to-google-sheets-14515


# Log invoice approval decisions from Slack to Google Sheets

## 1. Workflow Overview

This workflow listens for Slack interactive button clicks related to invoice approvals, extracts the decision and invoice metadata from the Slack payload, routes the execution based on the selected action, writes the invoice record into the corresponding Google Sheets tab, and finally sends a Slack direct message confirming the outcome.

Typical use case:
- A separate workflow posts an invoice review message in Slack with interactive buttons such as **Approve**, **Reject**, and **Flag**.
- When a user clicks one of those buttons, Slack sends an HTTP POST request to this workflow.
- This workflow logs the decision for downstream finance tracking and notifies the user.

### 1.1 Input Reception
Handles the incoming Slack interactivity webhook request.

### 1.2 Payload Parsing and Data Extraction
Parses Slack’s nested payload structure and extracts decision, approver identity, and invoice fields from the Slack message blocks.

### 1.3 Decision Routing
Uses a Switch node to branch the flow into Approved, Rejected, or Flagged paths.

### 1.4 Logging to Google Sheets
Appends the invoice data to one of three sheet tabs: `Approved`, `Rejected`, or `Flagged`.

### 1.5 Slack Confirmation Messages
Sends a direct Slack confirmation message after logging is complete.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block exposes an HTTP endpoint for Slack Interactivity. It is the workflow entry point and receives button click events as POST requests.

**Nodes Involved:**  
- Receive Slack Button Click

#### Node Details

##### Receive Slack Button Click
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Webhook trigger node that starts the workflow when Slack sends an interactive event.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `invoice-approval-handler`
  - No additional response handling options are configured
- **Key expressions or variables used:**  
  None in node configuration.
- **Input and output connections:**  
  - Input: none, this is the trigger
  - Output: `Parse Slack Payload`
- **Version-specific requirements:**  
  Uses webhook node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - Slack may reject the integration if the webhook URL is unreachable or inactive.
  - If Slack signs requests and validation is required, that is not implemented here.
  - If Slack sends a different body format than expected, downstream parsing will fail.
  - No explicit webhook response customization is present, so behavior depends on n8n defaults.
- **Sub-workflow reference:**  
  None.

---

### 2.2 Payload Parsing and Data Extraction

**Overview:**  
This block converts Slack’s URL-encoded interactive payload into JSON and then extracts the relevant business fields. It depends heavily on the exact structure of the Slack message and block layout.

**Nodes Involved:**  
- Parse Slack Payload
- Extract Decision & Invoice Data

#### Node Details

##### Parse Slack Payload
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a structured `payload` object by parsing the raw Slack payload string.
- **Configuration choices:**  
  - Assigns one field:
    - `payload` as an object using `JSON.parse($json.body.payload)`
- **Key expressions or variables used:**  
  - `={{ JSON.parse($json.body.payload) }}`
- **Input and output connections:**  
  - Input: `Receive Slack Button Click`
  - Output: `Extract Decision & Invoice Data`
- **Version-specific requirements:**  
  Uses Set node typeVersion `3.4`.
- **Edge cases or potential failure types:**  
  - If `body.payload` is missing, null, or malformed JSON, the expression throws an error.
  - If Slack payload format changes, this node may stop working.
- **Sub-workflow reference:**  
  None.

##### Extract Decision & Invoice Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts both decision metadata and invoice details from the parsed Slack payload.
- **Configuration choices:**  
  Assigns the following fields:
  - `decision` = selected button value
  - `decided_by` = Slack username
  - `decided_by_id` = Slack user ID
  - `channel_id` = Slack channel ID
  - `message_ts` = Slack message timestamp
  - `supplier_name` = extracted from block field text
  - `invoice_number` = extracted from block field text
  - `amount` = extracted from block field text
  - `invoice_date` = extracted from block field text
- **Key expressions or variables used:**  
  - `={{ $json.payload.actions[0].value }}`
  - `={{ $json.payload.user.name }}`
  - `={{ $json.payload.user.id }}`
  - `={{ $json.payload.channel.id }}`
  - `={{ $json.payload.message.ts }}`
  - `={{ $json.payload.message.blocks[1].fields[0].text.split('\n')[1] }}`
  - `={{ $json.payload.message.blocks[1].fields[1].text.split('\n')[1] }}`
  - `={{ $json.payload.message.blocks[1].fields[2].text.split('\n')[1] }}`
  - `={{ $json.payload.message.blocks[1].fields[3].text.split('\n')[1] }}`
- **Input and output connections:**  
  - Input: `Parse Slack Payload`
  - Output: `Route: Approved / Rejected / Flagged`
- **Version-specific requirements:**  
  Uses Set node typeVersion `3.4`.
- **Edge cases or potential failure types:**  
  - Assumes `actions[0]` exists and contains a valid `value`.
  - Assumes invoice data is always in `message.blocks[1].fields[0..3]`.
  - Assumes each block field contains a newline and that the desired value is after the first line.
  - Any Slack message formatting change in the originating workflow may break extraction.
  - If button values are not exactly `approved`, `rejected`, or `flagged`, routing will fail downstream.
- **Sub-workflow reference:**  
  None.

---

### 2.3 Decision Routing

**Overview:**  
This block branches execution into one of three paths based on the extracted decision. It is the control point that determines which Google Sheets tab and Slack confirmation message will be used.

**Nodes Involved:**  
- Route: Approved / Rejected / Flagged

#### Node Details

##### Route: Approved / Rejected / Flagged
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes one incoming item into a named output based on string comparisons.
- **Configuration choices:**  
  Three named outputs are defined:
  - `Approved` when `decision == approved`
  - `Rejected` when `decision == rejected`
  - `Flagged` when `decision == flagged`
  Output renaming is enabled for clarity.
- **Key expressions or variables used:**  
  - `={{ $json.decision }}`
- **Input and output connections:**  
  - Input: `Extract Decision & Invoice Data`
  - Output 1: `Log to Sheets: Approved`
  - Output 2: `Log to Sheets: Rejected`
  - Output 3: `Log to Sheets: Flagged`
- **Version-specific requirements:**  
  Uses Switch node typeVersion `3.2` with conditions options version `2`.
- **Edge cases or potential failure types:**  
  - If `decision` is missing or not one of the expected exact lowercase values, no branch will match.
  - Comparisons are case-sensitive with strict validation enabled.
- **Sub-workflow reference:**  
  None.

---

### 2.4 Logging to Google Sheets

**Overview:**  
This block writes invoice rows into the target spreadsheet. Each branch appends identical invoice fields, but to a different tab depending on the decision.

**Nodes Involved:**  
- Log to Sheets: Approved
- Log to Sheets: Rejected
- Log to Sheets: Flagged

#### Node Details

##### Log to Sheets: Approved
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to the `Approved` tab.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Approved`
  - Mapping mode: define below
  - Columns mapped:
    - `Supplier Name` ← `supplier_name`
    - `Invoice Number` ← `invoice_number`
    - `Amount` ← `amount`
    - `Date` ← `invoice_date`
- **Key expressions or variables used:**  
  - `={{ $json.invoice_date }}`
  - `={{ $json.amount }}`
  - `={{ $json.supplier_name }}`
  - `={{ $json.invoice_number }}`
- **Input and output connections:**  
  - Input: `Route: Approved / Rejected / Flagged` approved branch
  - Output: `Slack DM: Approved ✅`
- **Version-specific requirements:**  
  Uses Google Sheets node typeVersion `4.5`.
- **Edge cases or potential failure types:**  
  - Invalid or missing Google Sheets OAuth2 credentials.
  - Spreadsheet ID incorrect or inaccessible.
  - Sheet tab `Approved` missing.
  - Column headers in the sheet not matching configured schema can cause mapping issues.
- **Sub-workflow reference:**  
  None.

##### Log to Sheets: Rejected
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to the `Rejected` tab.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Rejected`
  - Same column mapping as the approved path
- **Key expressions or variables used:**  
  - `={{ $json.invoice_date }}`
  - `={{ $json.amount }}`
  - `={{ $json.supplier_name }}`
  - `={{ $json.invoice_number }}`
- **Input and output connections:**  
  - Input: `Route: Approved / Rejected / Flagged` rejected branch
  - Output: `Slack DM: Rejected ❌`
- **Version-specific requirements:**  
  Uses Google Sheets node typeVersion `4.5`.
- **Edge cases or potential failure types:**  
  Same as the Approved logging node, except for the `Rejected` tab requirement.
- **Sub-workflow reference:**  
  None.

##### Log to Sheets: Flagged
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to the `Flagged` tab.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Flagged`
  - Same column mapping as the approved path
- **Key expressions or variables used:**  
  - `={{ $json.invoice_date }}`
  - `={{ $json.amount }}`
  - `={{ $json.supplier_name }}`
  - `={{ $json.invoice_number }}`
- **Input and output connections:**  
  - Input: `Route: Approved / Rejected / Flagged` flagged branch
  - Output: `Slack DM: Flagged 🚩`
- **Version-specific requirements:**  
  Uses Google Sheets node typeVersion `4.5`.
- **Edge cases or potential failure types:**  
  Same as the Approved logging node, except for the `Flagged` tab requirement.
- **Sub-workflow reference:**  
  None.

---

### 2.5 Slack Confirmation Messages

**Overview:**  
This block sends a direct Slack message confirming the decision outcome. The message content uses the row data returned by the Google Sheets append operation and also references the approver name from the routing node context.

**Nodes Involved:**  
- Slack DM: Approved ✅
- Slack DM: Rejected ❌
- Slack DM: Flagged 🚩

#### Node Details

##### Slack DM: Approved ✅
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message to a specific user.
- **Configuration choices:**  
  - Recipient mode: user
  - Recipient ID: placeholder `YOUR_USER_ID`
  - Message text contains supplier, invoice number, amount, date, and approver name
- **Key expressions or variables used:**  
  - `{{ $json['Supplier Name'] }}`
  - `{{ $json['Invoice Number'] }}`
  - `{{ $json.Amount }}`
  - `{{ $json.Date }}`
  - `{{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}`
- **Input and output connections:**  
  - Input: `Log to Sheets: Approved`
  - Output: none
- **Version-specific requirements:**  
  Uses Slack node typeVersion `2.3`.
- **Edge cases or potential failure types:**  
  - Invalid Slack credentials or insufficient bot scopes.
  - User ID invalid or bot cannot message the user.
  - If the Google Sheets node does not return the expected column labels, message expressions may fail or be blank.
  - This sends to a fixed user ID, not dynamically to the approver.
- **Sub-workflow reference:**  
  None.

##### Slack DM: Rejected ❌
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a rejection confirmation message to a specific user.
- **Configuration choices:**  
  - Recipient mode: user
  - Recipient ID: placeholder `YOUR_USER_ID`
  - Message includes invoice details and rejecting user
- **Key expressions or variables used:**  
  - `{{ $json['Supplier Name'] }}`
  - `{{ $json['Invoice Number'] }}`
  - `{{ $json.Amount }}`
  - `{{ $json.Date }}`
  - `{{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}`
- **Input and output connections:**  
  - Input: `Log to Sheets: Rejected`
  - Output: none
- **Version-specific requirements:**  
  Uses Slack node typeVersion `2.3`.
- **Edge cases or potential failure types:**  
  Same as the approved DM node.
- **Sub-workflow reference:**  
  None.

##### Slack DM: Flagged 🚩
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a flagged/manual-review confirmation message to a specific user.
- **Configuration choices:**  
  - Recipient mode: user
  - Recipient ID: placeholder `YOUR_USER_ID`
  - Message includes invoice details, flagging user, and a manual review warning
- **Key expressions or variables used:**  
  - `{{ $json['Supplier Name'] }}`
  - `{{ $json['Invoice Number'] }}`
  - `{{ $json.Amount }}`
  - `{{ $json.Date }}`
  - `{{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}`
- **Input and output connections:**  
  - Input: `Log to Sheets: Flagged`
  - Output: none
- **Version-specific requirements:**  
  Uses Slack node typeVersion `2.3`.
- **Edge cases or potential failure types:**  
  Same as the approved DM node.
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Slack Button Click | Webhook | Receives Slack interactive POST requests |  | Parse Slack Payload | ### 🔔 Slack Webhook<br>Receives button click events from Slack Interactivity |
| Parse Slack Payload | Set | Parses Slack’s JSON payload from request body | Receive Slack Button Click | Extract Decision & Invoice Data | ### 📦 Parse Payload<br>Converts Slack's URL-encoded payload into JSON object |
| Extract Decision & Invoice Data | Set | Extracts decision, approver, and invoice fields | Parse Slack Payload | Route: Approved / Rejected / Flagged | ### 📋 Extract Decision Data<br>Pulls decision, user, and invoice details from Slack message |
| Route: Approved / Rejected / Flagged | Switch | Branches execution by decision value | Extract Decision & Invoice Data | Log to Sheets: Approved; Log to Sheets: Rejected; Log to Sheets: Flagged | ### 🔀 Route by Decision<br>Branches flow: Approved → Rejected → Flagged |
| Log to Sheets: Approved | Google Sheets | Appends approved invoice row to sheet | Route: Approved / Rejected / Flagged | Slack DM: Approved ✅ | ### 📊 Log to Sheets<br>Appends invoice data to the appropriate Google Sheets tab |
| Log to Sheets: Rejected | Google Sheets | Appends rejected invoice row to sheet | Route: Approved / Rejected / Flagged | Slack DM: Rejected ❌ | ### 📊 Log to Sheets<br>Appends invoice data to the appropriate Google Sheets tab |
| Log to Sheets: Flagged | Google Sheets | Appends flagged invoice row to sheet | Route: Approved / Rejected / Flagged | Slack DM: Flagged 🚩 | ### 📊 Log to Sheets<br>Appends invoice data to the appropriate Google Sheets tab |
| Slack DM: Approved ✅ | Slack | Sends approval confirmation DM | Log to Sheets: Approved |  | ### 💬 Send Confirmation<br>Notifies the approver via Slack DM |
| Slack DM: Rejected ❌ | Slack | Sends rejection confirmation DM | Log to Sheets: Rejected |  | ### 💬 Send Confirmation<br>Notifies the approver via Slack DM |
| Slack DM: Flagged 🚩 | Slack | Sends flagged confirmation DM | Log to Sheets: Flagged |  | ### 💬 Send Confirmation<br>Notifies the approver via Slack DM |
| Sticky Note | Sticky Note | Visual documentation for webhook block |  |  | # 📥 Invoice Approval Handler<br>(Slack Button Listener)<br><br>## What This Workflow Does<br>Listens for button clicks from the **Invoice → Slack Approval** workflow. When someone clicks Approve, Reject, or Flag on an invoice in Slack, this workflow captures that decision, logs it to **Google Sheets**, and sends a confirmation notification.<br><br>## How It Works<br>1. **Webhook Trigger** – Receives POST request from Slack when a button is clicked<br>2. **Parse Payload** – Extracts the JSON payload from Slack's request body<br>3. **Extract Decision Data** – Pulls decision, user info, and invoice details from the message<br>4. **Route by Decision** – Branches to Approved, Rejected, or Flagged path<br>5. **Log to Sheets** – Appends invoice data to the appropriate sheet tab<br>6. **Notify via Slack** – Sends confirmation DM to the approver<br><br>## Decision Routes<br>- ✅ **Approved** → Logs to "Approved" sheet → DM confirmation<br>- ❌ **Rejected** → Logs to "Rejected" sheet → DM confirmation<br>- 🚩 **Flagged** → Logs to "Flagged" sheet → DM for manual review<br><br>---<br><br>## Setup Guide<br><br>### 1. Create Google Sheet<br>1. Create a new Google Sheet with 3 tabs: `Approved`, `Rejected`, `Flagged`<br>2. Add these column headers to each tab:<br>   - Supplier Name<br>   - Invoice Number<br>   - Amount<br>   - Date<br>3. Copy the **Sheet ID** from the URL<br><br>### 2. Connect the Nodes in n8n<br>1. Add your **Google Sheets OAuth2** credential to all three logging nodes<br>2. Update the **Document ID** in each Google Sheets node to your Sheet ID<br>3. Add your **Slack API** credential to all three notification nodes<br>4. Update the **User ID** in the notification nodes (or change to channel)<br><br>### 3. Configure Slack Interactivity<br>1. Go to **api.slack.com/apps** → your app → **Interactivity & Shortcuts**<br>2. Set the Request URL to this workflow's webhook URL<br>3. Save Changes<br><br>### 4. Activate & Test<br>1. Click **Active** in the top-right corner of n8n<br>2. Trigger the Invoice Approval workflow to send a Slack message<br>3. Click a button in Slack<br>4. Check Google Sheets and Slack for results |
| Sticky Note1 | Sticky Note | Visual documentation for payload parsing block |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation for extraction block |  |  |  |
| Sticky Note3 | Sticky Note | Visual documentation for routing block |  |  |  |
| Sticky Note4 | Sticky Note | Visual documentation for sheet logging block |  |  |  |
| Sticky Note5 | Sticky Note | Visual documentation for Slack notification block |  |  |  |
| Sticky Note6 | Sticky Note | Global workflow documentation |  |  |  |

> Note: Sticky note nodes themselves are included because the requirement is not to skip any nodes. Their operational role is documentation only.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Invoice Approval Handler (Slack Listener)`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Receive Slack Button Click`
   - HTTP Method: `POST`
   - Path: `invoice-approval-handler`
   - Keep response options at default unless you want custom Slack acknowledgment handling.
   - This will be the entry point used by Slack Interactivity.

3. **Add a Set node to parse Slack payload**
   - Node type: `Set`
   - Name: `Parse Slack Payload`
   - Create one field:
     - Field name: `payload`
     - Type: `Object`
     - Value:
       ```n8n
       {{ JSON.parse($json.body.payload) }}
       ```
   - Connect `Receive Slack Button Click` → `Parse Slack Payload`.

4. **Add a Set node to extract decision and invoice details**
   - Node type: `Set`
   - Name: `Extract Decision & Invoice Data`
   - Add these fields:
     - `decision` as String:
       ```n8n
       {{ $json.payload.actions[0].value }}
       ```
     - `decided_by` as String:
       ```n8n
       {{ $json.payload.user.name }}
       ```
     - `decided_by_id` as String:
       ```n8n
       {{ $json.payload.user.id }}
       ```
     - `channel_id` as String:
       ```n8n
       {{ $json.payload.channel.id }}
       ```
     - `message_ts` as String:
       ```n8n
       {{ $json.payload.message.ts }}
       ```
     - `supplier_name` as String:
       ```n8n
       {{ $json.payload.message.blocks[1].fields[0].text.split('\n')[1] }}
       ```
     - `invoice_number` as String:
       ```n8n
       {{ $json.payload.message.blocks[1].fields[1].text.split('\n')[1] }}
       ```
     - `amount` as String:
       ```n8n
       {{ $json.payload.message.blocks[1].fields[2].text.split('\n')[1] }}
       ```
     - `invoice_date` as String:
       ```n8n
       {{ $json.payload.message.blocks[1].fields[3].text.split('\n')[1] }}
       ```
   - Connect `Parse Slack Payload` → `Extract Decision & Invoice Data`.

5. **Add a Switch node for decision routing**
   - Node type: `Switch`
   - Name: `Route: Approved / Rejected / Flagged`
   - Configure 3 outputs with renamed labels:
     1. `Approved`
        - Condition: `{{$json.decision}}` equals `approved`
     2. `Rejected`
        - Condition: `{{$json.decision}}` equals `rejected`
     3. `Flagged`
        - Condition: `{{$json.decision}}` equals `flagged`
   - Keep comparisons strict and case-sensitive if you want behavior identical to the source workflow.
   - Connect `Extract Decision & Invoice Data` → `Route: Approved / Rejected / Flagged`.

6. **Prepare the Google Sheet**
   - In Google Sheets, create one spreadsheet with 3 tabs:
     - `Approved`
     - `Rejected`
     - `Flagged`
   - Add the following header row to each tab:
     - `Supplier Name`
     - `Invoice Number`
     - `Amount`
     - `Date`
   - Copy the spreadsheet ID from the URL.

7. **Create the Approved logging node**
   - Node type: `Google Sheets`
   - Name: `Log to Sheets: Approved`
   - Credential: Google Sheets OAuth2
   - Operation: `Append`
   - Document ID: your spreadsheet ID
   - Sheet Name: `Approved`
   - Use manual field mapping / define-below mode
   - Map:
     - `Supplier Name` → `{{$json.supplier_name}}`
     - `Invoice Number` → `{{$json.invoice_number}}`
     - `Amount` → `{{$json.amount}}`
     - `Date` → `{{$json.invoice_date}}`
   - Connect the `Approved` output of the Switch node to this node.

8. **Create the Rejected logging node**
   - Node type: `Google Sheets`
   - Name: `Log to Sheets: Rejected`
   - Credential: same Google Sheets OAuth2 credential
   - Operation: `Append`
   - Document ID: same spreadsheet ID
   - Sheet Name: `Rejected`
   - Use the same column mapping as above
   - Connect the `Rejected` output of the Switch node to this node.

9. **Create the Flagged logging node**
   - Node type: `Google Sheets`
   - Name: `Log to Sheets: Flagged`
   - Credential: same Google Sheets OAuth2 credential
   - Operation: `Append`
   - Document ID: same spreadsheet ID
   - Sheet Name: `Flagged`
   - Use the same column mapping as above
   - Connect the `Flagged` output of the Switch node to this node.

10. **Create the Slack approval confirmation node**
    - Node type: `Slack`
    - Name: `Slack DM: Approved ✅`
    - Credential: Slack API credential
    - Send to: `User`
    - User ID: set a valid Slack user ID, or replace with a dynamic expression if you want to DM the actual clicker
    - Message text:
      ```n8n
      ✅ *Invoice Approved & Ready to Pay*

      *Supplier:* {{ $json['Supplier Name'] }}
      *Invoice #:* {{ $json['Invoice Number'] }}
      *Amount:* {{ $json.Amount }}
      *Date:* {{ $json.Date }}
      *Approved by:* {{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}
      ```
    - Connect `Log to Sheets: Approved` → `Slack DM: Approved ✅`.

11. **Create the Slack rejection confirmation node**
    - Node type: `Slack`
    - Name: `Slack DM: Rejected ❌`
    - Credential: same Slack credential
    - Send to: `User`
    - User ID: valid Slack user ID
    - Message text:
      ```n8n
      ❌ *Invoice Rejected*

      *Supplier:* {{ $json['Supplier Name'] }}
      *Invoice #:* {{ $json['Invoice Number'] }}
      *Amount:* {{ $json.Amount }}
      *Date:* {{ $json.Date }}
      *Rejected by:* {{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}
      ```
    - Connect `Log to Sheets: Rejected` → `Slack DM: Rejected ❌`.

12. **Create the Slack flagged confirmation node**
    - Node type: `Slack`
    - Name: `Slack DM: Flagged 🚩`
    - Credential: same Slack credential
    - Send to: `User`
    - User ID: valid Slack user ID
    - Message text:
      ```n8n
      🚩 *Invoice Flagged for Review*

      *Supplier:* {{ $json['Supplier Name'] }}
      *Invoice #:* {{ $json['Invoice Number'] }}
      *Amount:* {{ $json.Amount }}
      *Date:* {{ $json.Date }}
      *Flagged by:* {{ $('Route: Approved / Rejected / Flagged').item.json.decided_by }}

      ⚠️ Please review this invoice manually.
      ```
    - Connect `Log to Sheets: Flagged` → `Slack DM: Flagged 🚩`.

13. **Create credentials**
    - **Google Sheets OAuth2**
      - Must have access to the target spreadsheet.
    - **Slack API**
      - Must allow sending messages to users.
      - Depending on Slack app setup, typical scopes may include message posting scopes and user conversation access.
    - Ensure the Slack app is installed in the target workspace.

14. **Configure Slack Interactivity**
    - In Slack App settings:
      - Open `Interactivity & Shortcuts`
      - Enable Interactivity
      - Set the Request URL to the production webhook URL from `Receive Slack Button Click`
    - Save changes.
    - Make sure the Slack buttons in the upstream invoice message use action values:
      - `approved`
      - `rejected`
      - `flagged`

15. **Test with sample Slack message**
    - Trigger the upstream workflow that posts the invoice approval message.
    - Click each button in Slack.
    - Confirm:
      - the correct branch runs,
      - a row is appended to the correct sheet tab,
      - the Slack DM is sent.

16. **Recommended hardening improvements**
    - Add request validation for Slack signatures.
    - Add an explicit webhook response node or fast acknowledgment path if Slack expects immediate response timing.
    - Add fallback logic if block indexes or field positions differ.
    - Consider logging `decided_by`, `decided_by_id`, `channel_id`, and `message_ts` into Sheets as additional audit columns.
    - Consider sending the DM dynamically to `decided_by_id` instead of a fixed `YOUR_USER_ID`.

**Sub-workflow setup:**  
This workflow does not invoke any sub-workflow and is not configured as an Execute Workflow child flow. It does, however, depend on an upstream Slack message producer whose message block structure must remain compatible with the extraction expressions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed to work with a separate invoice-to-Slack approval flow that posts the interactive message. | Upstream dependency |
| Slack message parsing assumes the invoice data is located in `message.blocks[1].fields[0..3]`. Any change to the Slack message layout will require updating the Set expressions. | Implementation constraint |
| The Google Sheets nodes use a placeholder spreadsheet ID: `YOUR_GOOGLE_SHEET_ID`. | Replace with your actual spreadsheet ID |
| The Slack DM nodes use a placeholder Slack user ID: `YOUR_USER_ID`. | Replace with a real Slack user ID or make it dynamic |
| Google Sheets tabs required: `Approved`, `Rejected`, `Flagged`. | Spreadsheet structure |
| Suggested Slack configuration path: `api.slack.com/apps` → your app → `Interactivity & Shortcuts`. | Slack app settings |