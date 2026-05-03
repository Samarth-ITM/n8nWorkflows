Send vendor follow-up emails for overdue POs with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/send-vendor-follow-up-emails-for-overdue-pos-with-google-sheets-and-gmail-14440


# Send vendor follow-up emails for overdue POs with Google Sheets and Gmail

# 1. Workflow Overview

This workflow automates vendor follow-up emails for overdue purchase orders stored in Google Sheets. Every day at 9:00 AM, it reads a purchase order log and a vendor directory, identifies overdue POs that still require follow-up, matches each PO to vendor contact details, groups overdue POs per vendor into a single email, sends the email through Gmail, and updates the PO sheet with the current follow-up date.

Typical use cases:
- Procurement teams tracking overdue deliveries
- Operations teams managing supplier reminders
- Small businesses using Google Sheets as a lightweight PO tracker

## 1.1 Scheduled Input Reception
The workflow starts on a daily schedule and loads data from two separate Google Sheets documents:
- Purchase order log
- Vendor contact base

## 1.2 PO Validation and Date Normalization
The PO sheet rows are normalized and filtered so only actionable overdue items remain. This block also enforces the anti-spam rule: vendors should not be followed up again if they were contacted within the last 7 days.

## 1.3 Vendor Data Merge
Filtered PO records are joined with vendor records using `Vendor ID`, allowing the workflow to associate each overdue PO with the correct supplier email address.

## 1.4 Vendor-Level Email Composition
The merged records are grouped by vendor so that each vendor receives one consolidated message listing all overdue POs instead of multiple separate emails.

## 1.5 Outbound Communication and Tracking Update
The workflow sends the grouped emails through Gmail, while also updating the PO log sheet with today’s date in the `Last Follow-up Date` column for matched vendor records.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

**Overview:**  
This block launches the workflow every morning and retrieves raw data from the two source spreadsheets. It establishes the two parallel input streams later used for filtering and merging.

**Nodes Involved:**  
- Trigger
- Read PO
- Read Vendors

### Node: Trigger
- **Type and technical role:** `Schedule Trigger` node; entry point of the workflow
- **Configuration choices:** Configured to run at an interval rule with `triggerAtHour: 9`, which means the workflow executes daily at 9:00 AM according to the workflow/server timezone.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: none
  - Output: to `Read PO` and `Read Vendors`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Timezone mismatch between expected business timezone and n8n instance timezone
  - Workflow inactive: no executions occur until activated
- **Sub-workflow reference:** None

### Node: Read PO
- **Type and technical role:** `Google Sheets` node; reads purchase order rows from the PO log workbook
- **Configuration choices:**  
  - Uses Google Sheets OAuth2 credentials
  - Reads from spreadsheet named **Purchase Order log Book**
  - Targets sheet `gid=0` / `Sheet1`
  - No extra read options configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: from `Trigger`
  - Output: to `Filter + Normalize`
- **Version-specific requirements:** `typeVersion: 4`
- **Edge cases or potential failure types:**  
  - Google OAuth token expired or revoked
  - Spreadsheet not shared with the authenticated account
  - Sheet renamed, deleted, or moved
  - Missing expected columns such as `Vendor ID`, `Delivery Date`, `Delivery Status`, `Last Follow-up Date`, `PO Number`
- **Sub-workflow reference:** None

### Node: Read Vendors
- **Type and technical role:** `Google Sheets` node; reads vendor master/contact data
- **Configuration choices:**  
  - Uses the same Google Sheets OAuth2 credentials
  - Reads from spreadsheet named **Vendor Base**
  - Targets sheet `gid=0` / `Sheet1`
  - No extra read options configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: from `Trigger`
  - Output: to `Merge` on input 1
- **Version-specific requirements:** `typeVersion: 4`
- **Edge cases or potential failure types:**  
  - Same Google Sheets access issues as above
  - Missing vendor contact columns such as `Vendor ID`, `Supplier Email`, `Email`, `Supplier Name`, or `Vendor Name`
- **Sub-workflow reference:** None

---

## 2.2 PO Validation and Date Normalization

**Overview:**  
This block parses dates from the PO sheet and keeps only rows that are both overdue and still incomplete, while enforcing a minimum 7-day gap between follow-up attempts.

**Nodes Involved:**  
- Filter + Normalize

### Node: Filter + Normalize
- **Type and technical role:** `Code` node; transforms incoming PO rows and filters actionable records
- **Configuration choices:**  
  - Uses Luxon `DateTime`
  - Calculates:
    - `now = DateTime.now()`
    - `sevenDaysAgo = now.minus({ days: 7 })`
  - Attempts to parse delivery and follow-up dates using multiple formats:
    - ISO
    - `yyyy-MM-dd`
    - `dd/MM/yyyy`
    - `M/d/yyyy`
  - Adds normalized fields:
    - `deliveryDateParsed`
    - `followUpParsed`
  - Filters records based on:
    - valid delivery date exists
    - delivery date is due or overdue
    - delivery status is not `Complete`
    - no last follow-up date, or follow-up date older than 7 days
- **Key expressions or variables used:**  
  - `item.json['Delivery Date']`
  - `item.json['Last Follow-up Date']`
  - `item.json['Delivery Status']`
  - Luxon `DateTime.fromISO`, `DateTime.fromFormat`
- **Input and output connections:**  
  - Input: from `Read PO`
  - Output: to `Merge` on input 0
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Dates in unsupported formats will parse as invalid and rows will be excluded
  - Blank or malformed `Delivery Date` causes the row to be dropped
  - Status comparison is case-sensitive; values like `complete` or `COMPLETE` will not be treated as complete
  - Time-of-day effects may matter when comparing `d <= now`
  - If the runtime does not expose Luxon as expected in the Code node environment, parsing will fail
- **Sub-workflow reference:** None

---

## 2.3 Vendor Data Merge

**Overview:**  
This block joins overdue PO records with vendor records so that each PO line carries supplier contact information from the vendor base.

**Nodes Involved:**  
- Merge

### Node: Merge
- **Type and technical role:** `Merge` node; combines two input streams by matching fields
- **Configuration choices:**  
  - Mode: `combine`
  - Match field: `Vendor ID`
  - Input 0 receives filtered PO records
  - Input 1 receives vendor rows
- **Key expressions or variables used:**  
  - Match key: `Vendor ID`
- **Input and output connections:**  
  - Input 0: from `Filter + Normalize`
  - Input 1: from `Read Vendors`
  - Output: to `Group by Vendor` and `Update PO Sheet`
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - If `Vendor ID` is missing or inconsistent across sheets, records will not merge
  - String formatting mismatches such as extra whitespace or numeric/text differences can prevent matches
  - Duplicate vendor rows with the same `Vendor ID` may create unexpected result combinations
- **Sub-workflow reference:** None

---

## 2.4 Vendor-Level Email Composition

**Overview:**  
This block groups merged overdue PO records by vendor and generates one consolidated email payload per vendor, including recipient email, subject, body, and PO number list.

**Nodes Involved:**  
- Group by Vendor

### Node: Group by Vendor
- **Type and technical role:** `Code` node; aggregates rows and prepares outbound email content
- **Configuration choices:**  
  - Returns no items if no incoming data
  - Uses flexible field lookup for vendor email:
    - `supplierEmail`
    - `Supplier Email`
    - `Email`
  - Uses flexible field lookup for vendor name:
    - `supplierName`
    - `Supplier Name`
    - `Vendor Name`
    - fallback `Vendor`
  - Uses flexible field lookup for vendor ID:
    - `Vendor ID`
    - `vendorId`
  - Skips records missing email or vendor ID
  - Aggregates all POs per vendor into:
    - `email`
    - `poNumbers`
    - `subject`
    - `body`
  - Throws an explicit error if the final grouped result is empty:
    - `"No matches found. Check if 'Vendor ID' and 'Supplier Email' exist in the input data."`
- **Key expressions or variables used:**  
  - `data.supplierEmail || data['Supplier Email'] || data['Email']`
  - `data['Vendor ID'] || data.vendorId`
  - `data.supplierName || data['Supplier Name'] || data['Vendor Name'] || 'Vendor'`
  - PO line format: `• PO {poNum} | Due: {dueDate}`
- **Input and output connections:**  
  - Input: from `Merge`
  - Output: to `Send Email`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If merge output lacks vendor email fields, all rows may be skipped and the node throws an error
  - Duplicate or conflicting emails for the same vendor ID are not resolved; first encountered record effectively defines shared vendor metadata
  - Missing `PO Number` or `Delivery Date` can produce incomplete message lines
  - If there are simply no overdue records, the code currently returns `[]` only when `items.length === 0`; but if merged items exist and all are skipped due to missing email/vendor ID, it throws an error
- **Sub-workflow reference:** None

---

## 2.5 Outbound Communication and Tracking Update

**Overview:**  
This block sends the generated email to each vendor and writes today’s follow-up date back into the PO tracking spreadsheet. The two actions happen in parallel from the same merged dataset branch.

**Nodes Involved:**  
- Send Email
- Update PO Sheet

### Node: Send Email
- **Type and technical role:** `Gmail` node; sends the follow-up email through a Gmail account
- **Configuration choices:**  
  - Recipient: `={{ $json.email }}`
  - Subject: `={{ $json.subject }}`
  - Message body: `={{ $json.body }}`
  - No additional Gmail send options configured
- **Key expressions or variables used:**  
  - `$json.email`
  - `$json.subject`
  - `$json.body`
- **Input and output connections:**  
  - Input: from `Group by Vendor`
  - Output: none
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Gmail OAuth token expired or revoked
  - Gmail sending limits or quota restrictions
  - Invalid or blank recipient email
  - Gmail account security policies blocking automated sends
- **Sub-workflow reference:** None

### Node: Update PO Sheet
- **Type and technical role:** `Google Sheets` node; updates the purchase order spreadsheet after follow-up processing
- **Configuration choices:**  
  - Operation: `update`
  - Spreadsheet: **Purchase Order log Book**
  - Sheet: `gid=0` / `Sheet1`
  - Matching column: `Vendor ID`
  - Values written:
    - `Vendor ID = {{ $json['Vendor ID'] }}`
    - `Last Follow-up Date = {{ $today.format('yyyy-MM-dd') }}`
  - Mapping mode: defined manually below
  - Type conversion disabled
- **Key expressions or variables used:**  
  - `$json['Vendor ID']`
  - `$today.format('yyyy-MM-dd')`
- **Input and output connections:**  
  - Input: from `Merge`
  - Output: none
- **Version-specific requirements:** `typeVersion: 4`
- **Edge cases or potential failure types:**  
  - Matching only on `Vendor ID` means all rows for a vendor may be updated ambiguously, depending on Google Sheets node behavior and matched records
  - If multiple overdue POs exist for the same vendor, this update strategy may not reliably target only the intended PO rows
  - If `Vendor ID` is blank or mismatched, no update occurs
  - Spreadsheet schema changes can break mapping
  - `$today` formatting depends on workflow timezone/date context
- **Sub-workflow reference:** None

**Important implementation note:**  
The update logic matches on `Vendor ID`, not `PO Number` or row number. If your PO sheet contains multiple rows for the same vendor, the update may affect multiple records or may not behave as precisely as intended. For row-level tracking, matching on `PO Number` or `row_number` would usually be safer.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger | Schedule Trigger | Starts the workflow daily at 9:00 AM |  | Read PO, Read Vendors | 1. Data Retrieval<br>This section triggers the daily run and pulls the raw data from both your Purchase Order logs and your Vendor contact database. |
| Read PO | Google Sheets | Reads purchase order rows from the PO log sheet | Trigger | Filter + Normalize | 1. Data Retrieval<br>This section triggers the daily run and pulls the raw data from both your Purchase Order logs and your Vendor contact database. |
| Read Vendors | Google Sheets | Reads vendor contact records from the vendor base sheet | Trigger | Merge | 1. Data Retrieval<br>This section triggers the daily run and pulls the raw data from both your Purchase Order logs and your Vendor contact database. |
| Filter + Normalize | Code | Parses dates and filters overdue POs based on completion and follow-up recency | Read PO | Merge | 2. Validation & Merging<br>Here, the workflow filters out completed orders and checks the "7-day rule." It then joins the PO data with vendor emails using the Vendor ID. |
| Merge | Merge | Combines overdue PO rows with vendor records by Vendor ID | Filter + Normalize, Read Vendors | Group by Vendor, Update PO Sheet | 2. Validation & Merging<br>Here, the workflow filters out completed orders and checks the "7-day rule." It then joins the PO data with vendor emails using the Vendor ID. |
| Group by Vendor | Code | Groups rows per vendor and creates one consolidated email payload | Merge | Send Email | 3. Communication & Tracking<br>This final stage batches multiple POs into a single professional email and "checks the box" in your spreadsheet by logging the follow-up date. |
| Send Email | Gmail | Sends vendor follow-up emails | Group by Vendor |  | 3. Communication & Tracking<br>This final stage batches multiple POs into a single professional email and "checks the box" in your spreadsheet by logging the follow-up date. |
| Update PO Sheet | Google Sheets | Writes today's follow-up date back to the PO log | Merge |  | 3. Communication & Tracking<br>This final stage batches multiple POs into a single professional email and "checks the box" in your spreadsheet by logging the follow-up date. |
| Sticky Note | Sticky Note | Workspace documentation |  |  | How it works<br>Chasing vendors manually is a time-sink. This workflow automates the "nagging" so you don't have to. Every morning at 9:00 AM, the system scans your Purchase Order Log to find orders that are past their delivery date and not yet marked as "Complete."<br><br>To keep your vendors happy, it includes an "anti-spam" filter: it only sends a follow-up if the Last Follow-up Date is empty or more than 7 days old. Before sending, it merges your PO data with your Vendor Base to get the right email addresses. Finally, it groups all overdue POs for a single vendor into one consolidated email and updates your spreadsheet with today’s date so you don't double-email them tomorrow.<br><br>Setup<br>Google Sheets: Connect your Google account. In the Read PO and Read Vendors nodes, select your specific spreadsheets. Ensure your column headers (like Vendor ID, Delivery Date, and Supplier Email) match the names used in the Code nodes.<br><br>Gmail: Authenticate your Gmail account in the Send Email node.<br><br>Date Formats: The workflow uses Luxon to parse dates. If your spreadsheet uses a non-standard date format, you may need to adjust the parseDate function in the Filter + Normalize node.<br><br>Testing: Run the workflow manually once to ensure the "Update PO Sheet" node correctly writes back to your spreadsheet. |
| Sticky Note1 | Sticky Note | Workspace documentation for retrieval stage |  |  | 1. Data Retrieval<br>This section triggers the daily run and pulls the raw data from both your Purchase Order logs and your Vendor contact database. |
| Sticky Note2 | Sticky Note | Workspace documentation for validation and merge stage |  |  | 2. Validation & Merging<br>Here, the workflow filters out completed orders and checks the "7-day rule." It then joins the PO data with vendor emails using the Vendor ID. |
| Sticky Note3 | Sticky Note | Workspace documentation for communication and tracking stage |  |  | 3. Communication & Tracking<br>This final stage batches multiple POs into a single professional email and "checks the box" in your spreadsheet by logging the follow-up date. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automated Vendor Follow-up & PO Tracker`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run once per day at **09:00**
   - Confirm the instance timezone matches your business timezone
   - This is the main workflow entry point

3. **Add the first Google Sheets node for purchase orders**
   - Node type: `Google Sheets`
   - Name it: `Read PO`
   - Connect `Trigger -> Read PO`
   - Authenticate with **Google Sheets OAuth2**
   - Select the spreadsheet used as the PO log
   - Select the target tab, here `Sheet1` / `gid=0`
   - Leave additional options empty unless your environment requires special read settings

4. **Add the second Google Sheets node for vendor master data**
   - Node type: `Google Sheets`
   - Name it: `Read Vendors`
   - Connect `Trigger -> Read Vendors`
   - Use the same or another Google Sheets OAuth2 credential
   - Select the spreadsheet containing vendor contacts
   - Select the target tab, here `Sheet1` / `gid=0`

5. **Prepare the PO sheet structure**
   - Ensure the PO spreadsheet contains at least:
     - `PO Number`
     - `Vendor ID`
     - `Delivery Date`
     - `Delivery Status`
     - `Last Follow-up Date`
   - If column names differ, you must adjust the code later

6. **Prepare the vendor sheet structure**
   - Ensure the vendor spreadsheet contains:
     - `Vendor ID`
     - one email column such as `Supplier Email`, `Email`, or `supplierEmail`
     - optionally vendor name columns such as `Supplier Name` or `Vendor Name`

7. **Add a Code node for filtering and normalization**
   - Node type: `Code`
   - Name it: `Filter + Normalize`
   - Connect `Read PO -> Filter + Normalize`
   - Paste the following logic conceptually:
     - Get current date/time
     - Compute date 7 days ago
     - Parse `Delivery Date` and `Last Follow-up Date`
     - Support multiple date formats
     - Keep only rows where:
       - delivery date is valid and due
       - delivery status is not `Complete`
       - last follow-up is blank or older than 7 days
   - The implemented code in this workflow uses Luxon and creates:
     - `deliveryDateParsed`
     - `followUpParsed`

8. **Use this filtering logic in the Code node**
   - The parsing behavior should support:
     - ISO format
     - `yyyy-MM-dd`
     - `dd/MM/yyyy`
     - `M/d/yyyy`
   - The code should reject invalid delivery dates
   - It should compare dates against `DateTime.now()`

9. **Add a Merge node**
   - Node type: `Merge`
   - Name it: `Merge`
   - Connect:
     - `Filter + Normalize -> Merge` as input 0
     - `Read Vendors -> Merge` as input 1
   - Set mode to **Combine**
   - Set matching field to **Vendor ID**
   - This ensures each filtered PO row is enriched with vendor contact data

10. **Add a Code node to create vendor-level email payloads**
    - Node type: `Code`
    - Name it: `Group by Vendor`
    - Connect `Merge -> Group by Vendor`
    - Configure code to:
      - Group rows by `Vendor ID`
      - Resolve email from one of:
        - `supplierEmail`
        - `Supplier Email`
        - `Email`
      - Resolve vendor name from one of:
        - `supplierName`
        - `Supplier Name`
        - `Vendor Name`
        - default `Vendor`
      - Build a bullet list of overdue PO lines
      - Output one item per vendor with:
        - `email`
        - `poNumbers`
        - `subject`
        - `body`

11. **Use the same message pattern**
    - Subject:
      - `Follow-up: X overdue PO(s)`
    - Body:
      - `Dear {Vendor Name},`
      - blank line
      - `The following POs are overdue:`
      - bullet list like `• PO 123 | Due: 2024-05-10`
      - blank line
      - `Please share an update.`
      - blank line
      - `Thanks.`

12. **Preserve the error behavior in the grouping node**
    - If no items arrive at all, return an empty list
    - If items arrive but none contain both a valid vendor ID and vendor email, throw an explicit error
    - This helps surface bad merge or sheet schema issues

13. **Add a Gmail node**
    - Node type: `Gmail`
    - Name it: `Send Email`
    - Connect `Group by Vendor -> Send Email`
    - Authenticate with **Gmail OAuth2**
    - Configure:
      - To: `{{ $json.email }}`
      - Subject: `{{ $json.subject }}`
      - Message: `{{ $json.body }}`

14. **Configure Gmail credentials**
    - Use a Gmail account authorized to send external emails
    - Ensure OAuth consent and scopes are valid
    - Test by sending to a safe internal address first if needed

15. **Add a Google Sheets update node**
    - Node type: `Google Sheets`
    - Name it: `Update PO Sheet`
    - Connect `Merge -> Update PO Sheet`
    - Set operation to **Update**
    - Use the same PO spreadsheet as `Read PO`
    - Select the same worksheet/tab

16. **Configure the update mapping**
    - Matching column:
      - `Vendor ID`
    - Fields to write:
      - `Vendor ID = {{ $json['Vendor ID'] }}`
      - `Last Follow-up Date = {{ $today.format('yyyy-MM-dd') }}`

17. **Recreate the schema mapping**
    - In manual field mapping mode, expose at least:
      - `Vendor ID`
      - `Last Follow-up Date`
    - The original node schema also contained removed/display metadata for:
      - `PO Number`
      - `Delivery Date`
      - `Delivery Status`
      - `row_number`
   - These metadata are not functionally critical, but the actual columns must exist in the sheet

18. **Understand the update limitation before going live**
    - This workflow updates by `Vendor ID`
    - If a vendor has multiple PO rows, the update may not uniquely identify the exact row to change
    - For a more precise rebuild, consider matching by:
      - `PO Number`, or
      - `row_number`
    - If you want exact parity with the provided workflow, still use `Vendor ID`

19. **Optional: add sticky notes for workspace clarity**
    - Add a note describing the whole process:
      - automated daily scan
      - overdue PO detection
      - 7-day anti-spam check
      - vendor merge
      - grouped email send
      - follow-up date writeback
    - Add stage notes:
      - `1. Data Retrieval`
      - `2. Validation & Merging`
      - `3. Communication & Tracking`

20. **Test the workflow manually**
    - Run the workflow once with sample data
    - Verify:
      - overdue rows are selected correctly
      - completed rows are excluded
      - recent follow-ups are excluded
      - vendor emails merge correctly
      - grouped messages look correct
      - the PO sheet receives today’s date in `Last Follow-up Date`

21. **Validate date formats**
    - If your dates are not in:
      - ISO
      - `yyyy-MM-dd`
      - `dd/MM/yyyy`
      - `M/d/yyyy`
    - Then modify the parsing function in `Filter + Normalize`

22. **Activate the workflow**
    - Once tests succeed, activate the workflow so the schedule trigger runs daily

### Expected input/output behavior
- **Input from PO sheet:** one row per purchase order
- **Input from vendor sheet:** one row per vendor
- **Merged intermediate output:** one row per overdue PO enriched with vendor fields
- **Grouped output to Gmail:** one row per vendor email
- **Update behavior:** one or more PO sheet rows updated with current follow-up date depending on match behavior

### Required credentials
- **Google Sheets OAuth2**
  - Must have access to both spreadsheets
- **Gmail OAuth2**
  - Must be allowed to send email from the selected mailbox

### No sub-workflows
This workflow does not include any sub-workflows and does not invoke external n8n workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Chasing vendors manually is a time-sink. This workflow automates the follow-up process by checking overdue POs every morning at 9:00 AM. | General workflow purpose |
| Anti-spam behavior: a follow-up is sent only if `Last Follow-up Date` is empty or more than 7 days old. | Business rule |
| Before sending, the workflow merges PO data with the Vendor Base to get supplier email addresses. | Processing logic |
| Multiple overdue POs for the same vendor are consolidated into one email. | Communication design |
| After processing, the workflow updates the spreadsheet with today’s date so the same vendor is not emailed again immediately. | Tracking logic |
| Google Sheets setup: ensure column headers such as `Vendor ID`, `Delivery Date`, and `Supplier Email` match the names used in the Code nodes. | Configuration guidance |
| Gmail setup: authenticate the Gmail account in the `Send Email` node. | Credential guidance |
| Date parsing uses Luxon. If the sheet uses non-standard date formats, adjust the `parseDate` function in `Filter + Normalize`. | Technical note |
| Run the workflow manually once to verify that `Update PO Sheet` writes back correctly. | Testing guidance |