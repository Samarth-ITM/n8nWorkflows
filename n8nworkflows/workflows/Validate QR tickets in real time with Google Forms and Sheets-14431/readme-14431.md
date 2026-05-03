Validate QR tickets in real time with Google Forms and Sheets

https://n8nworkflows.xyz/workflows/validate-qr-tickets-in-real-time-with-google-forms-and-sheets-14431


# Validate QR tickets in real time with Google Forms and Sheets

# 1. Workflow Overview

This workflow validates QR-based event tickets in near real time using Google Sheets data, likely fed by Google Forms submissions or a scanning interface. When a new row is added to a Google Sheet, the workflow extracts the scanned QR code, looks up the corresponding ticket in a source sheet, and then decides whether the scan is valid. It writes the outcome back to a Google Sheet as a validation log.

The workflow is organized into four functional blocks:

## 1.1 Trigger and Latest Submission Selection
This block starts the workflow when a new row is added to a Google Sheet, then reduces the incoming dataset to the most recent item so downstream logic only processes one scan event.

## 1.2 Ticket Code Extraction and Source Lookup
This block maps the scanned QR value into a normalized field and searches the configured Google Sheet for a matching ticket code.

## 1.3 Validation Routing
This block compares the scanned QR code against the retrieved ticket row and routes execution into either a valid or invalid branch.

## 1.4 Result Logging
This block appends either a valid result row containing attendee details, or an invalid result row containing fallback placeholders and the scanned QR value.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Latest Submission Selection

### Overview
This block listens for new rows added to a Google Sheet and passes the latest detected item into the workflow. It ensures that only one row is processed downstream, even if multiple items are present in execution context.

### Nodes Involved
- Sheets Update Trigger
- Select Latest Item

### Node Details

#### Sheets Update Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`  
  Watches a Google Sheet for new row additions.
- **Configuration choices:**
  - Event: `rowAdded`
  - Polling mode: every minute
  - Spreadsheet and sheet are selected by ID placeholders:
    - `YOUR_SPREADSHEET_ID_HERE`
    - `YOUR_SHEET_NAME_HERE`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: `Select Latest Item`
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires Google Sheets Trigger OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token expiration or invalid Google credentials
  - Wrong spreadsheet ID or sheet ID/name mapping
  - Polling delays due to 1-minute interval
  - Trigger may behave unexpectedly if rows are inserted by automation in bursts
- **Sub-workflow reference:** None

#### Select Latest Item
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node intended to return the last or latest row from the incoming items.
- **Configuration choices:**
  - JavaScript code:
    - First returns the last item in the `items` array
    - Contains additional sort logic after the first `return`, but that code is unreachable
- **Key expressions or variables used:**
  - `items`
  - `rows[rows.length - 1]`
  - Unused/unreachable: sort by `Timestamp`
- **Input and output connections:**
  - Input: `Sheets Update Trigger`
  - Output: `Set Ticket Code Field`
- **Version-specific requirements:**
  - Uses code node version `2`
- **Edge cases or potential failure types:**
  - If `items` is empty, `rows[rows.length - 1]` becomes `undefined`
  - The timestamp sort logic never executes because of the earlier `return`
  - Assumes the last item is the newest; that may not hold if upstream ordering changes
- **Sub-workflow reference:** None

---

## 2.2 Ticket Code Extraction and Source Lookup

### Overview
This block standardizes the scanned QR code into a field named `kode_tiket` and uses it to search a Google Sheet for a matching ticket row.

### Nodes Involved
- Set Ticket Code Field
- Read Trigger Sheet Rows

### Node Details

#### Set Ticket Code Field
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds a normalized field used later in lookup and comparison logic.
- **Configuration choices:**
  - Creates one string field:
    - `kode_tiket = {{ $json['QR code'] }}`
  - `executeOnce` is enabled
- **Key expressions or variables used:**
  - `{{$json['QR code']}}`
- **Input and output connections:**
  - Input: `Select Latest Item`
  - Output: `Read Trigger Sheet Rows`
- **Version-specific requirements:**
  - Uses set node version `3.4`
- **Edge cases or potential failure types:**
  - If the incoming item has no `QR code` field, `kode_tiket` will be empty or undefined
  - `executeOnce` can be problematic if multiple items ever reach this node and distinct values are expected
- **Sub-workflow reference:** None

#### Read Trigger Sheet Rows
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet using a lookup filter to find a ticket by code.
- **Configuration choices:**
  - Spreadsheet and sheet are configured via placeholders
  - Filter:
    - Lookup column: `TIKET`
    - Lookup value: `{{ $json.kode_tiket }}`
  - `alwaysOutputData` is enabled
- **Key expressions or variables used:**
  - `{{$json.kode_tiket}}`
- **Input and output connections:**
  - Input: `Set Ticket Code Field`
  - Output: `Route by Conditional Rules`
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.6`
- **Edge cases or potential failure types:**
  - Invalid Google Sheets credentials
  - Wrong sheet selected, causing no match even for valid tickets
  - If multiple rows share the same `TIKET`, multiple items may be returned
  - Because `alwaysOutputData` is enabled, downstream logic may still execute even when no real match is found
- **Sub-workflow reference:** None

---

## 2.3 Validation Routing

### Overview
This block compares the scanned ticket code with the retrieved row’s `TIKET` field and sends execution either to a valid-result append branch or to an alternative invalid-processing branch.

### Nodes Involved
- Route by Conditional Rules

### Node Details

#### Route by Conditional Rules
- **Type and technical role:** `n8n-nodes-base.switch`  
  Performs rule-based branching using two named outputs.
- **Configuration choices:**
  - Output 1 renamed to `VALID`
  - Output 2 renamed to `TIDAK VALID`
  - Rule 1:
    - `$('Set Ticket Code Field').item.json.kode_tiket == $json.TIKET`
  - Rule 2:
    - `$('Set Ticket Code Field').item.json.kode_tiket != $json.TIKET`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `{{ $('Set Ticket Code Field').item.json.kode_tiket }}`
  - `{{ $json.TIKET }}`
- **Input and output connections:**
  - Input: `Read Trigger Sheet Rows`
  - Output 0 (`VALID`): `Append Scan Results`
  - Output 1 (`TIDAK VALID`): `Read Form Responses Sheet`
- **Version-specific requirements:**
  - Uses switch node version `3.2`
  - Rule options use condition version `2`
- **Edge cases or potential failure types:**
  - If lookup returned no row, `$json.TIKET` may be missing
  - If multiple rows are returned, each may be evaluated separately
  - Strict type validation may cause mismatches if one side is not a string
  - Continuing on error can hide underlying expression or data issues
- **Sub-workflow reference:** None

---

## 2.4 Result Logging

### Overview
This block writes validation outcomes back to Google Sheets. Valid tickets append attendee metadata and a `VALID` status, while invalid scans append placeholder values and a `TIDAK VALID` status.

### Nodes Involved
- Append Scan Results
- Read Form Responses Sheet
- Select Latest Form Response
- Append to Scan Results Sheet

### Node Details

#### Append Scan Results
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row for a valid ticket scan.
- **Configuration choices:**
  - Operation: `append`
  - Mapped columns:
    - `ID = {{ $json.ID }}`
    - `NAMA = {{ $json.NAMA }}`
    - `EMAIL = {{ $json.EMAIL }}`
    - `NO HP = {{ $json['NO HP'] }}`
    - `TIKET = {{ $json.TIKET }}`
    - `STATUS = VALID`
    - `UKURAN JERSEY = {{ $json['UKURAN JERSEY'] }}`
    - `GOLONGAN DARAH = {{ $json['GOLONGAN DARAH'] }}`
  - Appends into configured target sheet
- **Key expressions or variables used:**
  - Standard field references from matched ticket row
- **Input and output connections:**
  - Input: `Route by Conditional Rules` via `VALID`
  - Output: none
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.6`
- **Edge cases or potential failure types:**
  - Missing columns in destination sheet
  - Data shape mismatch if lookup source columns differ
  - Duplicate scans are not prevented by workflow logic
- **Sub-workflow reference:** None

#### Read Form Responses Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads an entire Google Sheet in the invalid branch.
- **Configuration choices:**
  - No filter configured
  - Reads from configured spreadsheet/sheet placeholders
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Route by Conditional Rules` via `TIDAK VALID`
  - Output: `Select Latest Form Response`
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.6`
- **Edge cases or potential failure types:**
  - Reads all rows, which may become inefficient on large sheets
  - No filter means the data is not actually tied to the invalid QR code
  - Wrong source sheet may lead to unrelated latest response being selected
- **Sub-workflow reference:** None

#### Select Latest Form Response
- **Type and technical role:** `n8n-nodes-base.code`  
  Selects the last item from the form response sheet.
- **Configuration choices:**
  - Same code pattern as `Select Latest Item`
  - Also contains unreachable sort logic after an early return
- **Key expressions or variables used:**
  - `items`
  - `rows[rows.length - 1]`
- **Input and output connections:**
  - Input: `Read Form Responses Sheet`
  - Output: `Append to Scan Results Sheet`
- **Version-specific requirements:**
  - Uses code node version `2`
- **Edge cases or potential failure types:**
  - Empty sheet results in undefined output
  - Unreachable timestamp logic means latest-by-time is not guaranteed
  - The selected row is not used meaningfully in the next node except as execution carrier
- **Sub-workflow reference:** None

#### Append to Scan Results Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends an invalid scan result row.
- **Configuration choices:**
  - Operation: `append`
  - Mapped columns:
    - `ID = -`
    - `NAMA = -`
    - `EMAIL = -`
    - `NO HP = -`
    - `TIKET = {{ $json['QR code'] }}`
    - `STATUS = TIDAK VALID`
    - `UKURAN JERSEY = -`
    - `GOLONGAN DARAH = -`
- **Key expressions or variables used:**
  - `{{$json['QR code']}}`
- **Input and output connections:**
  - Input: `Select Latest Form Response`
  - Output: none
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.6`
- **Edge cases or potential failure types:**
  - Likely expression issue: the incoming item from `Select Latest Form Response` may not contain `QR code`
  - This can cause the invalid ticket value to be blank unless data is explicitly carried forward or referenced from another node
  - Destination schema must contain all mapped columns
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation / overview note |  |  | ## Automated QR Ticket Scanner & Validator (Google Workspace)<br>### How it works<br>1. Triggers on new data in Google Sheets.<br>2. Processes the last added item via code execution.<br>3. Edits fields for ticket verification purposes.<br>4. Switches logic based on validation results.<br>5. Appends the result back to Google Sheets.<br>### Setup steps<br>- [ ] Set up Google Sheets API credentials for n8n.<br>- [ ] Configure the Google Sheets Trigger node with appropriate spreadsheet ID and trigger conditions.<br>- [ ] Ensure code nodes are configured appropriately to handle data logic.<br>### Customization<br>Users can customize the code logic nodes to fit different ticket validation criteria. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | ## Trigger and initial processing<br>Handles the trigger from Google Sheets and initial processing of data. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for lookup block |  |  | ## Edit fields and validation<br>Edits fields for verification and retrieves rows for processing. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for valid branch |  |  | ## Append results to sheet<br>Processes data after switching logic and appends results back to Google Sheets. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for invalid branch |  |  | ## Alternative processing and result appending<br>Handles alternative logic flows and appends different results based on secondary criteria. |
| Sheets Update Trigger | n8n-nodes-base.googleSheetsTrigger | Entry point when a new row is added to a Google Sheet |  | Select Latest Item | ## Trigger and initial processing<br>Handles the trigger from Google Sheets and initial processing of data. |
| Select Latest Item | n8n-nodes-base.code | Reduces incoming data to the last item | Sheets Update Trigger | Set Ticket Code Field | ## Trigger and initial processing<br>Handles the trigger from Google Sheets and initial processing of data. |
| Set Ticket Code Field | n8n-nodes-base.set | Maps scanned QR value into `kode_tiket` | Select Latest Item | Read Trigger Sheet Rows | ## Edit fields and validation<br>Edits fields for verification and retrieves rows for processing. |
| Read Trigger Sheet Rows | n8n-nodes-base.googleSheets | Looks up ticket row by ticket code | Set Ticket Code Field | Route by Conditional Rules | ## Edit fields and validation<br>Edits fields for verification and retrieves rows for processing. |
| Route by Conditional Rules | n8n-nodes-base.switch | Routes to valid or invalid branch based on ticket match | Read Trigger Sheet Rows | Append Scan Results; Read Form Responses Sheet | ## Edit fields and validation<br>Edits fields for verification and retrieves rows for processing. |
| Append Scan Results | n8n-nodes-base.googleSheets | Appends valid ticket scan result | Route by Conditional Rules |  | ## Append results to sheet<br>Processes data after switching logic and appends results back to Google Sheets. |
| Read Form Responses Sheet | n8n-nodes-base.googleSheets | Reads rows for invalid branch processing | Route by Conditional Rules | Select Latest Form Response | ## Alternative processing and result appending<br>Handles alternative logic flows and appends different results based on secondary criteria. |
| Select Latest Form Response | n8n-nodes-base.code | Selects last row from form responses sheet | Read Form Responses Sheet | Append to Scan Results Sheet | ## Alternative processing and result appending<br>Handles alternative logic flows and appends different results based on secondary criteria. |
| Append to Scan Results Sheet | n8n-nodes-base.googleSheets | Appends invalid ticket scan result | Select Latest Form Response |  | ## Alternative processing and result appending<br>Handles alternative logic flows and appends different results based on secondary criteria. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automated QR Ticket Scanner & Validator (Google Workspace)`.
   - Leave it inactive until credentials and sheets are verified.

2. **Add a Google Sheets Trigger node**
   - Node type: **Google Sheets Trigger**
   - Name: `Sheets Update Trigger`
   - Event: `Row Added`
   - Polling interval: every minute
   - Select Google credentials:
     - Create or choose a Google OAuth2 credential with access to the spreadsheet
   - Set:
     - `Document ID` = your spreadsheet ID
     - `Sheet Name` = the sheet receiving scans or Google Form entries

3. **Add a Code node after the trigger**
   - Node type: **Code**
   - Name: `Select Latest Item`
   - Connect: `Sheets Update Trigger -> Select Latest Item`
   - Use code equivalent to:
     ```javascript
     const rows = items;
     const lastRow = rows[rows.length - 1];
     return [lastRow];
     ```
   - Note: this reproduces the current workflow behavior.  
     If you want true latest-by-timestamp behavior, rewrite the code instead of copying the unreachable sort logic.

4. **Add a Set node**
   - Node type: **Set**
   - Name: `Set Ticket Code Field`
   - Connect: `Select Latest Item -> Set Ticket Code Field`
   - Add one field:
     - Name: `kode_tiket`
     - Type: `String`
     - Value: `{{ $json['QR code'] }}`
   - Keep other incoming fields if needed, depending on your n8n Set node behavior/settings.
   - Enable `Execute Once` to match the original workflow.

5. **Add a Google Sheets node for ticket lookup**
   - Node type: **Google Sheets**
   - Name: `Read Trigger Sheet Rows`
   - Connect: `Set Ticket Code Field -> Read Trigger Sheet Rows`
   - Credentials: same Google credential or another authorized one
   - Select the spreadsheet and sheet containing the master ticket list
   - Configure it to read/filter rows where:
     - Lookup column: `TIKET`
     - Lookup value: `{{ $json.kode_tiket }}`
   - Enable `Always Output Data`

6. **Add a Switch node**
   - Node type: **Switch**
   - Name: `Route by Conditional Rules`
   - Connect: `Read Trigger Sheet Rows -> Route by Conditional Rules`
   - Add two rule outputs and rename them:
     - Output 1: `VALID`
     - Output 2: `TIDAK VALID`
   - Rule 1 condition:
     - Left value: `{{ $('Set Ticket Code Field').item.json.kode_tiket }}`
     - Operator: `equals`
     - Right value: `{{ $json.TIKET }}`
   - Rule 2 condition:
     - Left value: `{{ $('Set Ticket Code Field').item.json.kode_tiket }}`
     - Operator: `notEquals`
     - Right value: `{{ $json.TIKET }}`
   - Enable error behavior equivalent to continuing regular output if available.
   - Enable output even when data is empty if you want parity with the original design.

7. **Build the valid branch append node**
   - Node type: **Google Sheets**
   - Name: `Append Scan Results`
   - Connect the `VALID` output from `Route by Conditional Rules` to this node
   - Operation: `Append`
   - Select the destination spreadsheet and sheet for scan logs
   - Map columns manually:
     - `ID` = `{{ $json.ID }}`
     - `NAMA` = `{{ $json.NAMA }}`
     - `EMAIL` = `{{ $json.EMAIL }}`
     - `NO HP` = `{{ $json['NO HP'] }}`
     - `TIKET` = `{{ $json.TIKET }}`
     - `STATUS` = `VALID`
     - `UKURAN JERSEY` = `{{ $json['UKURAN JERSEY'] }}`
     - `GOLONGAN DARAH` = `{{ $json['GOLONGAN DARAH'] }}`

8. **Build the invalid branch sheet read node**
   - Node type: **Google Sheets**
   - Name: `Read Form Responses Sheet`
   - Connect the `TIDAK VALID` output from `Route by Conditional Rules` to this node
   - Configure it to read from the form responses or related source sheet
   - No filter is configured in the original workflow

9. **Add another Code node**
   - Node type: **Code**
   - Name: `Select Latest Form Response`
   - Connect: `Read Form Responses Sheet -> Select Latest Form Response`
   - Use code equivalent to:
     ```javascript
     const rows = items;
     const lastRow = rows[rows.length - 1];
     return [lastRow];
     ```
   - This reproduces the current behavior.

10. **Add the invalid append node**
    - Node type: **Google Sheets**
    - Name: `Append to Scan Results Sheet`
    - Connect: `Select Latest Form Response -> Append to Scan Results Sheet`
    - Operation: `Append`
    - Select the destination sheet for validation logs
    - Map columns manually:
      - `ID` = `-`
      - `NAMA` = `-`
      - `EMAIL` = `-`
      - `NO HP` = `-`
      - `TIKET` = `{{ $json['QR code'] }}`
      - `STATUS` = `TIDAK VALID`
      - `UKURAN JERSEY` = `-`
      - `GOLONGAN DARAH` = `-`

11. **Verify field names in all sheets**
    - The workflow assumes exact column names such as:
      - `QR code`
      - `TIKET`
      - `ID`
      - `NAMA`
      - `EMAIL`
      - `NO HP`
      - `UKURAN JERSEY`
      - `GOLONGAN DARAH`
      - `Timestamp`
    - Field names are case-sensitive in practice when used in expressions.

12. **Configure credentials**
    - Create a Google OAuth2 credential in n8n with access to:
      - the trigger sheet
      - the master ticket sheet
      - the scan results sheet
      - any form response sheet used in the invalid branch
    - Ensure the Google account has edit permission for append operations.

13. **Test the valid path**
    - Add a new row into the trigger sheet with a `QR code` that exists in the master ticket sheet under `TIKET`
    - Confirm that a row is appended with status `VALID`

14. **Test the invalid path**
    - Add a new row with a `QR code` that does not exist in the lookup sheet
    - Confirm that a row is appended with status `TIDAK VALID`
    - Check whether the `TIKET` field is actually populated in the destination row

15. **Recommended correction for the invalid branch**
    - The current design likely loses the original `QR code` before reaching `Append to Scan Results Sheet`
    - To make it reliable, either:
      - pass the original scan data along with the invalid branch, or
      - reference it explicitly in the append node using:
        `{{ $('Set Ticket Code Field').item.json.kode_tiket }}`
    - This is not in the source workflow, but it is strongly recommended.

16. **Recommended correction for both Code nodes**
    - The provided code includes unreachable logic after `return`
    - If you want real latest-by-time behavior, use:
      ```javascript
      items.sort((a, b) => new Date(a.json.Timestamp) - new Date(b.json.Timestamp));
      return [items[items.length - 1]];
      ```
    - Only do this if `Timestamp` exists consistently and is parseable.

17. **Activate the workflow**
    - Once tests pass and sheet mappings are verified, activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow description and node comments indicate a Google Workspace-based ticket validation flow using Google Sheets as both trigger source and result log. | General architecture |
| The main note suggests that users can customize the code logic nodes to support different validation criteria. | Customization guidance |
| The trigger operates by polling every minute rather than using an instant webhook-style event. | Operational behavior |
| The current implementation does not include duplicate-scan prevention, attendee check-in state management, or ticket consumption tracking. | Functional limitation |
| The current invalid branch appears structurally weak because it reads another sheet and then appends using a `QR code` field that may no longer exist in the item context. | Design caveat |
| No external branding links, videos, blogs, or project credits were included in the workflow metadata or sticky notes. | Resource audit |