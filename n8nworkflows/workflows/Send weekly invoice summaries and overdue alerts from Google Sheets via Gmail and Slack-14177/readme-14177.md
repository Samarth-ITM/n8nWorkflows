Send weekly invoice summaries and overdue alerts from Google Sheets via Gmail and Slack

https://n8nworkflows.xyz/workflows/send-weekly-invoice-summaries-and-overdue-alerts-from-google-sheets-via-gmail-and-slack-14177


# Send weekly invoice summaries and overdue alerts from Google Sheets via Gmail and Slack

# 1. Workflow Overview

This workflow sends a weekly invoice summary email and, when applicable, posts an overdue-invoice alert to Slack. It is designed for teams that maintain invoice tracking data in Google Sheets and want automated reporting plus proactive overdue follow-up.

Typical use cases:
- Weekly finance reporting for small businesses
- Accounts receivable monitoring from a spreadsheet
- Slack escalation when unpaid invoices become overdue

The workflow is organized into five logical blocks.

## 1.1 Scheduled Trigger

The workflow starts automatically on a recurring schedule. In the current configuration, it runs at 9:00 AM every Monday.

## 1.2 Runtime Configuration

A Set node defines the operational settings used downstream:
- Google Sheet ID
- Sheet/tab name
- Email recipient
- Slack channel identifier
- Business name

This makes the workflow easy to adapt without changing multiple nodes.

## 1.3 Invoice Data Retrieval

The workflow reads all rows from a Google Sheets worksheet that stores invoice records. The sheet is expected to contain fields such as invoice ID, client name, amount, due date, and status.

## 1.4 Totals Calculation and Message Construction

A Code node processes every invoice row, computes paid/unpaid/overdue totals, identifies overdue invoices, builds an HTML summary for email, and prepares plain-text Slack alert content.

## 1.5 Reporting and Conditional Alerting

The workflow always sends the weekly summary by Gmail. It then evaluates whether overdue invoices exist; if so, it sends a Slack alert with invoice-level details.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger

### Overview
This block is the workflow entry point. It launches the process on a recurring weekly schedule and feeds one item into the configuration stage.

### Nodes Involved
- Every Monday 9 AM

### Node Details

#### Every Monday 9 AM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node for time-based workflow execution.
- **Configuration choices:**
  - Configured with an hourly trigger time of 9.
  - Based on the embedded note, the intended schedule is every Monday at 9 AM.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**
  - Input: none, as this is a trigger node.
  - Output: `Configure Settings`
- **Version-specific requirements:**
  - Node type version: `1.2`
  - Schedule Trigger behavior can vary slightly across n8n versions, especially in recurrence UI and timezone handling.
- **Edge cases or potential failure types:**
  - Timezone mismatch between server timezone and business expectation
  - Misconfigured recurrence if imported into another n8n version
  - Workflow inactive, so no runs occur
- **Sub-workflow reference:**  
  None

---

## 2.2 Runtime Configuration

### Overview
This block centralizes the core settings required by the workflow. It injects static configuration values into the data stream so downstream nodes can use expressions rather than hardcoded values.

### Nodes Involved
- Configure Settings

### Node Details

#### Configure Settings
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates and assigns configuration fields into the current item.
- **Configuration choices:**
  - Assigns five string fields:
    - `googleSheetId`
    - `sheetName`
    - `recipientEmail`
    - `slackChannel`
    - `businessName`
  - Current placeholder/default values:
    - `YOUR_GOOGLE_SHEET_ID`
    - `Invoices`
    - `user@example.com`
    - `YOUR_SLACK_CHANNEL_ID`
    - `Your Business Name`
- **Key expressions or variables used:**  
  No expressions inside the Set node; it produces variables used later.
- **Input and output connections:**
  - Input: `Every Monday 9 AM`
  - Output: `Read All Invoices`
- **Version-specific requirements:**
  - Node type version: `3.4`
  - The Set node UI differs across n8n versions, but assignment behavior is standard.
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged
  - Invalid Google Sheet ID
  - Invalid recipient email
  - Slack channel setting is defined here but not actually used by the Slack node as currently configured
- **Sub-workflow reference:**  
  None

---

## 2.3 Invoice Data Retrieval

### Overview
This block reads all invoice rows from the target Google Sheet tab. It depends on the configuration values prepared upstream.

### Nodes Involved
- Read All Invoices

### Node Details

#### Read All Invoices
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Fetches rows from a Google Sheets document.
- **Configuration choices:**
  - Sheet name is dynamically read from `{{$json.sheetName}}`
  - Document ID is dynamically read from `{{$json.googleSheetId}}`
  - No extra options are configured
  - The node appears intended to read all rows from the specified worksheet
- **Key expressions or variables used:**
  - `={{ $json.sheetName }}`
  - `={{ $json.googleSheetId }}`
- **Input and output connections:**
  - Input: `Configure Settings`
  - Output: `Calculate Totals and Find Overdue`
- **Version-specific requirements:**
  - Node type version: `4.5`
  - Requires valid Google Sheets credentials in n8n
  - Depending on n8n version, explicit operation/mode may default differently; verify after import
- **Edge cases or potential failure types:**
  - Google authentication failure
  - Spreadsheet not found
  - Worksheet/tab name mismatch
  - Missing header row or inconsistent column names
  - Empty sheet producing zero invoice rows
  - Rate limiting or transient Google API errors
- **Sub-workflow reference:**  
  None

---

## 2.4 Totals Calculation and Message Construction

### Overview
This is the core logic block. It reads all invoice rows, classifies them by status and due date, calculates totals, generates the HTML email body, and prepares Slack-formatted overdue text.

### Nodes Involved
- Calculate Totals and Find Overdue

### Node Details

#### Calculate Totals and Find Overdue
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript processing node for aggregation and content generation.
- **Configuration choices:**
  - Reads all incoming invoice rows via `$input.all().map(i => i.json)`
  - Pulls the configuration item directly from the `Configure Settings` node using `$('Configure Settings').first().json`
  - Initializes totals:
    - `totalPaid`
    - `totalUnpaid`
    - `totalOverdue`
  - Builds two lists:
    - `overdueList`
    - `unpaidList`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$('Configure Settings').first().json`
  - Field fallbacks:
    - `inv['Amount'] || inv['amount'] || 0`
    - `inv['Status'] || inv['status'] || ''`
    - `inv['Due Date'] || inv['due_date'] || ''`
    - `inv['Client Name'] || inv['client_name'] || 'Unknown'`
    - `inv['Invoice ID'] || inv['invoice_id'] || 'N/A'`
- **Processing logic:**
  - Converts amount using `parseFloat`
  - Lowercases invoice status
  - Parses due date with `new Date(...)`
  - If status is `paid`, amount contributes to `totalPaid`
  - Otherwise:
    - amount contributes to `totalUnpaid`
    - invoice enters `unpaidList`
    - if `dueDate < today`, it also contributes to `totalOverdue` and enters `overdueList`
  - Generates:
    - `emailHtml`
    - `slackOverdueText`
    - formatted totals
    - `overdueCount`
    - `hasOverdue`
    - `reportDate`
- **Input and output connections:**
  - Input: `Read All Invoices`
  - Outputs:
    - `Email Weekly Summary`
    - `Any Overdue?`
- **Version-specific requirements:**
  - Node type version: `2`
  - Requires Code node support in your n8n version
- **Edge cases or potential failure types:**
  - **Invalid date handling:** `new Date('')` produces an invalid date. Calling `toISOString()` on an invalid date will throw a runtime error.
  - **Date timezone drift:** comparing `dueDate < today` includes time-of-day, so invoices due today may be treated inconsistently depending on timezones and midnight offsets.
  - **Status mismatch:** any non-`paid` status is treated as unpaid. That includes blank, malformed, or unexpected values.
  - **Amount parsing:** non-numeric amounts may become `NaN`, which can contaminate totals.
  - **Column naming inconsistency:** only a limited set of alternate column names is supported.
  - **HTML escaping:** invoice or client values are inserted into HTML without escaping.
  - **Slack formatting issues:** special characters in names or IDs may affect message formatting.
- **Sub-workflow reference:**  
  None

---

## 2.5 Reporting and Conditional Alerting

### Overview
This block sends the summary email every run and conditionally sends a Slack alert only if overdue invoices exist.

### Nodes Involved
- Email Weekly Summary
- Any Overdue?
- Alert Overdue to Slack

### Node Details

#### Email Weekly Summary
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email via Gmail.
- **Configuration choices:**
  - Recipient: `{{$json.recipientEmail}}`
  - Subject: `Weekly Invoice Summary - {{ $json.businessName }} ({{ $json.reportDate }})`
  - Message body: `{{$json.emailHtml}}`
  - No extra options configured
- **Key expressions or variables used:**
  - `={{ $json.recipientEmail }}`
  - `=Weekly Invoice Summary - {{ $json.businessName }} ({{ $json.reportDate }})`
  - `={{ $json.emailHtml }}`
- **Input and output connections:**
  - Input: `Calculate Totals and Find Overdue`
  - Output: none
- **Version-specific requirements:**
  - Node type version: `2.1`
  - Requires Gmail credentials configured in n8n
  - Confirm whether the message field is treated as HTML in your installed node version; some setups may require explicit HTML/body options
- **Edge cases or potential failure types:**
  - Gmail authentication error
  - Sending limits or quota restrictions
  - HTML rendering differences
  - Invalid recipient address
  - Email may still send even if the Slack branch later fails, because both branches execute independently after the Code node
- **Sub-workflow reference:**  
  None

#### Any Overdue?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branch node that checks whether overdue invoices exist.
- **Configuration choices:**
  - Case-insensitive condition
  - Evaluates boolean equality:
    - left value: `{{$json.hasOverdue}}`
    - right value: `true`
- **Key expressions or variables used:**
  - `={{ $json.hasOverdue }}`
- **Input and output connections:**
  - Input: `Calculate Totals and Find Overdue`
  - True output: `Alert Overdue to Slack`
  - False output: none
- **Version-specific requirements:**
  - Node type version: `2.2`
- **Edge cases or potential failure types:**
  - If `hasOverdue` is missing or not a real boolean, the condition may fail unexpectedly
  - No false-branch action means non-overdue runs end silently after email
- **Sub-workflow reference:**  
  None

#### Alert Overdue to Slack
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message to a channel.
- **Configuration choices:**
  - Authentication: OAuth2
  - Message text contains:
    - warning header
    - business name
    - overdue invoice count
    - total overdue amount
    - line-by-line overdue invoice details
  - Channel selection is set to a specific channel by name:
    - `#general`
  - Although the Set node defines `slackChannel`, this node does not use that value
- **Key expressions or variables used:**
  - `=:warning: *Overdue Invoice Alert* - {{ $json.businessName }}
{{ $json.overdueCount }} invoice(s) overdue, totalling *${{ $json.totalOverdueFormatted }}*

{{ $json.slackOverdueText }}`
- **Input and output connections:**
  - Input: `Any Overdue?` true branch
  - Output: none
- **Version-specific requirements:**
  - Node type version: `2.4`
  - Requires Slack OAuth2 credentials with permission to post to the target channel
- **Edge cases or potential failure types:**
  - Slack auth failure
  - Bot not invited to the channel
  - Channel selection mismatch
  - Message length limits if many overdue invoices exist
  - Current configuration ignores `slackChannel` from settings, which may confuse maintainers
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Monday 9 AM | Schedule Trigger | Starts the workflow on a weekly schedule |  | Configure Settings | ## 1. Weekly trigger<br>Fires every Monday at 9 AM. Edit the schedule if you want it daily or biweekly. |
| Configure Settings | Set | Defines reusable runtime settings | Every Monday 9 AM | Read All Invoices | ## 2. Your settings<br>Sheet ID, recipient email, Slack channel, business name. Change these and you're good to go. |
| Read All Invoices | Google Sheets | Reads all invoice rows from the configured spreadsheet tab | Configure Settings | Calculate Totals and Find Overdue | ## 3. Pull the data<br>Grabs every row from your invoice spreadsheet. |
| Calculate Totals and Find Overdue | Code | Computes totals, identifies overdue invoices, builds email HTML and Slack text | Read All Invoices | Email Weekly Summary; Any Overdue? | ## 4. Crunch the numbers<br>Adds up paid, unpaid, and overdue totals. Builds the email HTML with a table of overdue invoices and how many days late each one is. |
| Email Weekly Summary | Gmail | Sends the weekly invoice summary email | Calculate Totals and Find Overdue |  | ## 5. Email report + Slack alert<br>The full summary gets emailed. If any invoices are overdue, Slack gets a separate alert with the specifics. |
| Any Overdue? | If | Checks whether overdue invoices exist | Calculate Totals and Find Overdue | Alert Overdue to Slack | ## 5. Email report + Slack alert<br>The full summary gets emailed. If any invoices are overdue, Slack gets a separate alert with the specifics. |
| Alert Overdue to Slack | Slack | Posts overdue invoice details to Slack when needed | Any Overdue? |  | ## 5. Email report + Slack alert<br>The full summary gets emailed. If any invoices are overdue, Slack gets a separate alert with the specifics. |
| Sticky Note | Sticky Note | Canvas documentation and setup instructions |  |  | ### Send weekly invoice summary with overdue alerts from Google Sheets<br><br>If you track invoices in a spreadsheet but nobody checks it until cash flow gets tight, this gives you an automatic weekly report with the numbers that actually matter. It also pings Slack when invoices are overdue so nothing slips through.<br><br>### How it works<br>1. Runs every Monday at 9 AM (you can change the schedule)<br>2. Reads all rows from your invoice tracker in Google Sheets<br>3. A code node crunches the numbers: total paid, total unpaid, total overdue. It also builds an HTML email with a proper breakdown table.<br>4. The summary report gets emailed to whoever you configure<br>5. If anything is overdue, a Slack message goes out with the details so your team can chase it<br><br>### Setup<br>1. Create a Google Sheet with an "Invoices" tab. Columns: Invoice ID, Client Name, Amount, Due Date, Status (paid/unpaid/overdue)<br>2. Open "Configure Settings" and add your Sheet ID, recipient email, Slack channel ID, and business name<br>3. Connect your Google Sheets, Gmail, and Slack credentials<br>4. Activate and wait for Monday morning (or trigger it manually to test) |
| Sticky Note1 | Sticky Note | Visual annotation for trigger block |  |  | ## 1. Weekly trigger<br>Fires every Monday at 9 AM. Edit the schedule if you want it daily or biweekly. |
| Sticky Note2 | Sticky Note | Visual annotation for settings block |  |  | ## 2. Your settings<br>Sheet ID, recipient email, Slack channel, business name. Change these and you're good to go. |
| Sticky Note3 | Sticky Note | Visual annotation for Google Sheets read block |  |  | ## 3. Pull the data<br>Grabs every row from your invoice spreadsheet. |
| Sticky Note4 | Sticky Note | Visual annotation for calculation block |  |  | ## 4. Crunch the numbers<br>Adds up paid, unpaid, and overdue totals. Builds the email HTML with a table of overdue invoices and how many days late each one is. |
| Sticky Note5 | Sticky Note | Visual annotation for reporting block |  |  | ## 5. Email report + Slack alert<br>The full summary gets emailed. If any invoices are overdue, Slack gets a separate alert with the specifics. |

---

# 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n.

## Prerequisites
1. Prepare a Google Sheet with a worksheet named `Invoices`.
2. Add a header row with these columns:
   - `Invoice ID`
   - `Client Name`
   - `Amount`
   - `Due Date`
   - `Status`
3. Ensure invoice statuses are normalized, ideally using:
   - `paid`
   - `unpaid`
   - `overdue`  
   Note: in this workflow, any status other than `paid` is treated as unpaid, and overdue is determined from due date.
4. Create and connect credentials in n8n for:
   - Google Sheets
   - Gmail
   - Slack OAuth2

## Build Steps

1. **Create a Schedule Trigger node**
   - Name it: `Every Monday 9 AM`
   - Type: `Schedule Trigger`
   - Configure it to run weekly, Monday at 9:00 AM
   - Verify the timezone used by your n8n instance

2. **Create a Set node**
   - Name it: `Configure Settings`
   - Type: `Set`
   - Add these string fields:
     - `googleSheetId` = your spreadsheet ID
     - `sheetName` = `Invoices`
     - `recipientEmail` = destination email address
     - `slackChannel` = your Slack channel ID
     - `businessName` = your company or organization name
   - Connect `Every Monday 9 AM` → `Configure Settings`

3. **Create a Google Sheets node**
   - Name it: `Read All Invoices`
   - Type: `Google Sheets`
   - Select Google Sheets credentials
   - Configure it to read rows from a spreadsheet
   - Set **Document ID** with an expression:
     - `{{$json.googleSheetId}}`
   - Set **Sheet Name** with an expression:
     - `{{$json.sheetName}}`
   - Set it to return all rows from the worksheet
   - Connect `Configure Settings` → `Read All Invoices`

4. **Create a Code node**
   - Name it: `Calculate Totals and Find Overdue`
   - Type: `Code`
   - Language: JavaScript
   - Paste logic that:
     - collects all rows from input
     - retrieves config from `Configure Settings`
     - parses amount, status, due date, client name, invoice ID
     - totals paid, unpaid, overdue
     - builds `unpaidList` and `overdueList`
     - generates an HTML string for email
     - generates a Slack text string for overdue invoices
     - returns one output item containing the computed fields
   - The output item should include at minimum:
     - `recipientEmail`
     - `businessName`
     - `emailHtml`
     - `totalPaid`
     - `totalUnpaid`
     - `totalOverdue`
     - formatted totals
     - `overdueCount`
     - `hasOverdue`
     - `slackOverdueText`
     - `reportDate`
   - Connect `Read All Invoices` → `Calculate Totals and Find Overdue`

5. **Create a Gmail node**
   - Name it: `Email Weekly Summary`
   - Type: `Gmail`
   - Select Gmail credentials
   - Configure send operation
   - Set **To**:
     - `{{$json.recipientEmail}}`
   - Set **Subject**:
     - `Weekly Invoice Summary - {{$json.businessName}} ({{$json.reportDate}})`
   - Set **Message**:
     - `{{$json.emailHtml}}`
   - If your Gmail node version requires explicit HTML/body settings, enable HTML body mode accordingly
   - Connect `Calculate Totals and Find Overdue` → `Email Weekly Summary`

6. **Create an If node**
   - Name it: `Any Overdue?`
   - Type: `If`
   - Configure one condition:
     - Left value: `{{$json.hasOverdue}}`
     - Operator: boolean equals
     - Right value: `true`
   - Connect `Calculate Totals and Find Overdue` → `Any Overdue?`

7. **Create a Slack node**
   - Name it: `Alert Overdue to Slack`
   - Type: `Slack`
   - Select Slack OAuth2 credentials
   - Configure it to send a channel message
   - Set **Text** to something like:
     - `:warning: *Overdue Invoice Alert* - {{$json.businessName}}`
     - `{{$json.overdueCount}} invoice(s) overdue, totalling *${{$json.totalOverdueFormatted}}*`
     - `{{$json.slackOverdueText}}`
   - Choose the target channel
   - Important: the source workflow stores `slackChannel` in the Set node, but the Slack node is currently hardcoded to `#general`. To reproduce exactly, use a fixed channel. To improve it, bind the channel dynamically to the config value.
   - Connect `Any Overdue?` true output → `Alert Overdue to Slack`

8. **Add optional sticky notes**
   - Add notes to document:
     - purpose of the workflow
     - trigger timing
     - settings to edit
     - spreadsheet assumptions
     - reporting behavior

9. **Test the workflow**
   - Execute manually
   - Confirm:
     - Google Sheets returns rows
     - totals are computed correctly
     - email renders correctly
     - Slack only posts when overdue invoices exist

10. **Activate the workflow**
   - Turn on the workflow after credentials and test data are verified

## Credential Configuration Details

### Google Sheets
- Use a Google account with access to the spreadsheet
- Ensure the spreadsheet is shared with that account if needed
- Verify the Sheet ID matches the URL of the spreadsheet

### Gmail
- Use Gmail OAuth2 credentials
- Make sure the authenticated account is allowed to send emails to the configured recipient
- Check Gmail daily sending limits if used at scale

### Slack OAuth2
- Use a Slack app/bot with permission to post messages
- Ensure the bot has access to the target channel
- If posting to a private channel, invite the bot explicitly

## Recommended Hardening Improvements While Rebuilding
If you want a more production-safe version, apply these adjustments:
1. Validate due dates before calling `toISOString()`
2. Normalize dates to local midnight before overdue comparison
3. Treat explicit `overdue` status separately if your spreadsheet relies on status rather than date logic
4. Escape HTML content for invoice and client fields
5. Use the configured `slackChannel` value in the Slack node instead of a fixed channel
6. Add an error handling branch or an Error Trigger workflow
7. Add a node to handle empty-sheet cases gracefully

## Sub-workflow Setup
- This workflow does **not** use any sub-workflows.
- It has a single entry point: `Every Monday 9 AM`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Send weekly invoice summary with overdue alerts from Google Sheets | Workflow purpose |
| Runs every Monday at 9 AM, but the schedule can be changed | Trigger behavior |
| Expected worksheet name is `Invoices` | Google Sheets setup |
| Expected columns: Invoice ID, Client Name, Amount, Due Date, Status | Spreadsheet structure |
| Configure Sheet ID, recipient email, Slack channel ID, and business name in the Set node | Initial customization |
| Connect Google Sheets, Gmail, and Slack credentials before activation | Operational requirement |
| Activate the workflow or trigger it manually for testing | Deployment/testing guidance |

## Additional Implementation Notes
- The workflow contains a configuration field named `slackChannel`, but the Slack node is currently set to post to `#general` by name. If you need true configurability, update the Slack node to use the configured value.
- The Code node determines overdue status from due date comparison, not from the spreadsheet `Status` value alone.
- The summary email is always sent, even when there are no overdue invoices.
- The Slack alert is only sent when `hasOverdue` is true.