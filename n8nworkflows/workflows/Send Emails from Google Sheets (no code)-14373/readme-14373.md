Send Emails from Google Sheets (no code)

https://n8nworkflows.xyz/workflows/send-emails-from-google-sheets--no-code--14373


# Send Emails from Google Sheets (no code)

## 1. Workflow Overview

This workflow manages a simple email outreach process from Google Sheets and also records incoming replies back into the same spreadsheet.

It is split into two independent branches:

- **Manual sending branch**: started manually to send all emails whose status is marked as `To send`
- **Automatic reply tracking branch**: runs continuously via Gmail polling to capture incoming replies and save them into the sheet

The spreadsheet acts as the system of record. Each row represents one outbound email and stores:

- recipient email
- subject
- message body
- send status
- optional response text

### 1.1 Outbound Email Sending

This block starts when a user manually runs the workflow. It fetches rows from Google Sheets where `Status = To send`, sends each email through Gmail, then updates the same row to `Sent`.

### 1.2 Incoming Reply Monitoring

This block watches the connected Gmail inbox every minute. When a new email arrives, it attempts to match the sender’s address to a row in the sheet and only selects rows where the `Response` column is still empty. It then writes the email text into that row’s `Response` field.

### 1.3 Documentation and Context Notes

The workflow also contains sticky notes with operational guidance, creator contact information, and a video walkthrough link. These notes do not affect execution but are important for setup and maintenance.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Outbound Email Sending

**Overview:**  
This branch sends pending emails listed in a Google Sheet. It retrieves rows marked for sending, sends one Gmail message per row, and marks each processed row as sent to avoid duplicates.

**Nodes Involved:**  
- Manual Trigger - Start Email Campaign
- Get Pending Emails from Sheet
- Send Email via Gmail
- Mark Email as Sent

### Node: Manual Trigger - Start Email Campaign

- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Entry node for manual execution.
- **Configuration choices:**  
  No parameters are configured. The workflow starts only when triggered manually from the n8n editor or execution UI.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get Pending Emails from Sheet`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - No automatic scheduling; if forgotten, emails are not sent
  - User may run it multiple times, but duplicate sends are partly prevented by filtering on `Status = To send`
- **Sub-workflow reference:**  
  None.

### Node: Get Pending Emails from Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads spreadsheet rows that match a filter.
- **Configuration choices:**  
  - Uses a Google Sheets OAuth2 credential named `sheets account`
  - Targets spreadsheet `Email Manager`
  - Targets sheet/tab `Emails` (`gid=0`)
  - Filters rows where column `Status` equals `To send`
- **Key expressions or variables used:**  
  No dynamic expression in the filter value; it uses a fixed value: `To send`
- **Input and output connections:**  
  - Input: `Manual Trigger - Start Email Campaign`
  - Output: `Send Email via Gmail`
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Invalid or expired Google Sheets OAuth2 credential
  - Spreadsheet or sheet access denied
  - Missing `Status` column
  - No matching rows, in which case the branch simply produces no items
  - Sheet structure changes may break downstream assumptions about `To`, `Subject`, `Message`, and `row_number`
- **Sub-workflow reference:**  
  None.

### Node: Send Email via Gmail

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends outbound email messages using Gmail.
- **Configuration choices:**  
  - Uses Gmail OAuth2 credential `john.doe@example.com`
  - `sendTo` is taken from `{{$json.To}}`
  - `subject` is taken from `{{$json.Subject}}`
  - `message` is taken from `{{$json.Message}}`
  - Email type is `text`
  - `appendAttribution` is disabled, so n8n branding is not appended
- **Key expressions or variables used:**  
  - `={{ $json.To }}`
  - `={{ $json.Subject }}`
  - `={{ $json.Message }}`
- **Input and output connections:**  
  - Input: `Get Pending Emails from Sheet`
  - Output: `Mark Email as Sent`
- **Version-specific requirements:**  
  Type version `2.2`
- **Edge cases or potential failure types:**  
  - Invalid Gmail OAuth2 credential or revoked consent
  - Gmail sending limits or quota issues
  - Invalid recipient email address in `To`
  - Empty `Subject` or `Message`
  - If sending fails for a row, the following update node will not run for that item
- **Sub-workflow reference:**  
  None.

### Node: Mark Email as Sent

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates an existing row after successful sending.
- **Configuration choices:**  
  - Operation: `update`
  - Uses the same spreadsheet and sheet as the read step
  - Matches rows using `row_number`
  - Writes:
    - `Status = Sent`
    - `row_number = {{ $('Get Pending Emails from Sheet').item.json.row_number }}`
  - Mapping mode is explicitly defined
- **Key expressions or variables used:**  
  - `={{ $('Get Pending Emails from Sheet').item.json.row_number }}`
- **Input and output connections:**  
  - Input: `Send Email via Gmail`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Missing `row_number` from the source row
  - Spreadsheet row deleted or reordered between read and update
  - Permission issues on the sheet
  - If `row_number` is not unique or unavailable, update may fail or affect the wrong row
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: Incoming Reply Monitoring

**Overview:**  
This branch polls Gmail every minute for new incoming emails. It uses the sender address to find the original row in Google Sheets and stores the plain text reply in the `Response` column.

**Nodes Involved:**  
- On New Email Received
- Find Original Email Row
- Save Email Response

### Node: On New Email Received

- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Trigger node that polls Gmail for new messages.
- **Configuration choices:**  
  - Uses Gmail OAuth2 credential `john.doe@example.com`
  - Polling interval: every minute
  - `simple` mode is disabled, so the node returns a richer Gmail payload
  - No explicit filters are configured
- **Key expressions or variables used:**  
  None in parameters, but downstream nodes use fields from this output:
  - `$json.from.value[0].address`
  - `$json.text`
- **Input and output connections:**  
  - Input: none
  - Output: `Find Original Email Row`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Expired Gmail OAuth2 credential
  - Polling delays or missed expectations due to Gmail API behavior
  - Incoming messages without expected parsed sender structure
  - Replies from addresses not present in the sheet
  - Since no filters are configured, unrelated incoming emails can trigger lookup attempts
- **Sub-workflow reference:**  
  None.

### Node: Find Original Email Row

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Searches the sheet for the row corresponding to the incoming email sender.
- **Configuration choices:**  
  - Uses Google Sheets OAuth2 credential `sheets account`
  - Searches in spreadsheet `Email Manager`, sheet `Emails`
  - Filter 1: `To` equals `={{ $json.from.value[0].address }}`
  - Filter 2: `Response` column is also referenced with no lookup value configured, which effectively means the node is intended to find rows where response is blank/unset
- **Key expressions or variables used:**  
  - `={{ $json.from.value[0].address }}`
- **Input and output connections:**  
  - Input: `On New Email Received`
  - Output: `Save Email Response`
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - If Gmail output does not contain `from.value[0].address`, expression resolution may fail
  - If multiple rows exist for the same recipient, matching may be ambiguous
  - If the `Response` filter is not interpreted as intended, rows may not match as expected
  - Missing `To` or `Response` columns in the sheet
  - No matching row means no reply gets stored
- **Sub-workflow reference:**  
  None.

### Node: Save Email Response

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the matched row to store the incoming email body.
- **Configuration choices:**  
  - Operation: `update`
  - Matches on `row_number`
  - Writes:
    - `Response = {{ $('On New Email Received').item.json.text }}`
    - `row_number = {{ $json.row_number }}`
  - Uses explicit field mapping
- **Key expressions or variables used:**  
  - `={{ $('On New Email Received').item.json.text }}`
  - `={{ $json.row_number }}`
- **Input and output connections:**  
  - Input: `Find Original Email Row`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`
- **Edge cases or potential failure types:**  
  - Empty email body or no `text` field on incoming message
  - HTML-only emails may not produce the desired text output
  - Missing `row_number`
  - Permission/update failures in Google Sheets
  - If several messages arrive from the same sender, later ones may overwrite earlier response data
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: Documentation and Context Notes

**Overview:**  
These sticky notes provide setup instructions, creator contact details, and a video walkthrough. They are non-executable but useful for maintainers.

**Nodes Involved:**  
- Workflow Description
- Creator Contact Info
- Video Walkthrough

### Node: Workflow Description

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation node.
- **Configuration choices:**  
  Contains a long operational note describing purpose, setup, and configuration expectations.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None for execution.
- **Sub-workflow reference:**  
  None.

### Node: Creator Contact Info

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation with creator branding and contact information.
- **Configuration choices:**  
  Includes links to booking, YouTube, LinkedIn, and an email address.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None for execution.
- **Sub-workflow reference:**  
  None.

### Node: Video Walkthrough

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation with an external video link.
- **Configuration choices:**  
  Includes a linked thumbnail pointing to YouTube.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None for execution.
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger - Start Email Campaign | Manual Trigger | Starts outbound email processing manually |  | Get Pending Emails from Sheet | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Get Pending Emails from Sheet | Google Sheets | Reads rows where `Status = To send` | Manual Trigger - Start Email Campaign | Send Email via Gmail | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Send Email via Gmail | Gmail | Sends outbound email for each selected row | Get Pending Emails from Sheet | Mark Email as Sent | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Mark Email as Sent | Google Sheets | Updates sent row status to `Sent` | Send Email via Gmail |  | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| On New Email Received | Gmail Trigger | Polls Gmail for incoming emails |  | Find Original Email Row | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Find Original Email Row | Google Sheets | Finds the matching spreadsheet row for an incoming email | On New Email Received | Save Email Response | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Save Email Response | Google Sheets | Writes inbound email text into the `Response` column | Find Original Email Row |  | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Workflow Description | Sticky Note | Embedded operational documentation |  |  | ## Workflow Overview This workflow automates repetitive email sending directly from Google Sheets and tracks responses, helping you reclaim hours spent on manual emailing. Simply maintain your email list in a spreadsheet with columns for recipient, subject, message, and status—then let n8n handle the rest. |
| Creator Contact Info | Sticky Note | Creator branding and support information |  |  | # Contact Us: Milan @ SmoothWork - [Book a Free Consulting Call](https://smoothwork.ai/book-a-call/)  / We help businesses eliminate busywork by building compact business tools tailored to your process. / Contact us for customizing this, or building similar automations. / hello@smoothwork.ai / [Check us on YouTube](https://www.youtube.com/@vasarmilan) / [Book a Free Consulting Call](https://smoothwork.ai/book-a-call/) / [Add me on Linkedin](https://www.linkedin.com/in/mil%C3%A1n-v%C3%A1s%C3%A1rhelyi-3a9985123/) |
| Video Walkthrough | Sticky Note | Linked video reference |  |  | # Video Walkthrough / [https://www.youtube.com/watch?v=jxT6XO4eUwI](https://www.youtube.com/watch?v=jxT6XO4eUwI) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet**
   1. Create a spreadsheet for email management.
   2. Add a sheet/tab, for example `Emails`.
   3. Add these columns in the first row:
      - `To`
      - `Subject`
      - `Message`
      - `Status`
      - `Response`
   4. Add sample rows.
   5. Use `To send` in the `Status` column for emails that should be sent.

2. **Create a new workflow in n8n**
   1. Open n8n.
   2. Create a blank workflow.
   3. Give it the title: **Send Emails from Google Sheets (no code)**.

3. **Add the manual outbound branch entry node**
   1. Add a **Manual Trigger** node.
   2. Rename it to **Manual Trigger - Start Email Campaign**.
   3. Leave default settings unchanged.

4. **Add the Google Sheets read node for pending emails**
   1. Add a **Google Sheets** node.
   2. Rename it to **Get Pending Emails from Sheet**.
   3. Connect **Manual Trigger - Start Email Campaign** → **Get Pending Emails from Sheet**.
   4. Configure Google Sheets credentials using **Google Sheets OAuth2**.
   5. Select your spreadsheet document.
   6. Select the correct sheet/tab.
   7. Configure the node to read/filter rows.
   8. In filters, add:
      - Column: `Status`
      - Value: `To send`
   9. Ensure the node returns row metadata including `row_number` so later updates can target the same row.

5. **Add the Gmail sending node**
   1. Add a **Gmail** node.
   2. Rename it to **Send Email via Gmail**.
   3. Connect **Get Pending Emails from Sheet** → **Send Email via Gmail**.
   4. Configure Gmail OAuth2 credentials for the mailbox that will send the emails.
   5. Set the operation to send email.
   6. Configure:
      - **To**: `={{ $json.To }}`
      - **Subject**: `={{ $json.Subject }}`
      - **Message**: `={{ $json.Message }}`
   7. Set email type to **Text**.
   8. Disable **Append Attribution**.

6. **Add the Google Sheets update node to mark sent emails**
   1. Add another **Google Sheets** node.
   2. Rename it to **Mark Email as Sent**.
   3. Connect **Send Email via Gmail** → **Mark Email as Sent**.
   4. Set operation to **Update**.
   5. Use the same spreadsheet and sheet as before.
   6. Configure matching column:
      - `row_number`
   7. Map fields explicitly:
      - `Status` = `Sent`
      - `row_number` = `={{ $('Get Pending Emails from Sheet').item.json.row_number }}`
   8. Keep type conversion disabled unless your sheet structure requires it.

7. **Add the inbound monitoring branch entry node**
   1. Add a **Gmail Trigger** node.
   2. Rename it to **On New Email Received**.
   3. Configure Gmail OAuth2 credentials using the same mailbox or another mailbox receiving replies.
   4. Set polling to **every minute**.
   5. Set **Simple** to **false** so full message data is available.
   6. Leave filters empty unless you want to narrow the watched inbox.

8. **Add the Google Sheets lookup node for reply matching**
   1. Add a **Google Sheets** node.
   2. Rename it to **Find Original Email Row**.
   3. Connect **On New Email Received** → **Find Original Email Row**.
   4. Select the same spreadsheet and sheet.
   5. Configure filters:
      - `To` = `={{ $json.from.value[0].address }}`
      - Add a second condition referencing `Response` to only target rows that do not yet have a stored response
   6. Verify behavior in your n8n version, because empty-value filtering in Google Sheets nodes can behave differently depending on configuration and version. If needed, replace this with a different filtering strategy.

9. **Add the Google Sheets update node to save the reply**
   1. Add another **Google Sheets** node.
   2. Rename it to **Save Email Response**.
   3. Connect **Find Original Email Row** → **Save Email Response**.
   4. Set operation to **Update**.
   5. Use the same spreadsheet and sheet.
   6. Match on:
      - `row_number`
   7. Map fields:
      - `Response` = `={{ $('On New Email Received').item.json.text }}`
      - `row_number` = `={{ $json.row_number }}`
   8. Save the node.

10. **Optional: add visual notes**
   1. Add a **Sticky Note** named **Workflow Description** and paste setup/usage instructions.
   2. Add a **Sticky Note** named **Creator Contact Info** if you want support or branding information visible in the canvas.
   3. Add a **Sticky Note** named **Video Walkthrough** with the video link.

11. **Credential setup requirements**
   1. **Google Sheets OAuth2**
      - Must have read and update access to the spreadsheet
      - The authenticated account must be able to open and edit the sheet
   2. **Gmail OAuth2**
      - Must have permission to send email
      - Must have permission to read mailbox content for the trigger
   3. If using separate Google accounts for Gmail and Sheets, ensure both credentials are configured independently.

12. **Test the outbound branch**
   1. Put one or more rows in the sheet with:
      - valid `To`
      - non-empty `Subject`
      - non-empty `Message`
      - `Status = To send`
   2. Execute the workflow manually.
   3. Confirm:
      - emails are sent
      - `Status` becomes `Sent`

13. **Test the inbound branch**
   1. Activate the workflow.
   2. Reply from one of the recipient email addresses.
   3. Wait for the Gmail Trigger polling interval.
   4. Confirm the matching row gets updated in `Response`.

14. **Recommended hardening improvements**
   1. Add validation for missing `To`, `Subject`, or `Message`.
   2. Add an IF node before sending to skip malformed rows.
   3. Add deduplication logic for multiple sheet rows with the same `To` value.
   4. Add a timestamp column such as `Sent At` or `Response At`.
   5. Add error handling paths for Gmail or Google Sheets failures.
   6. Consider matching replies using thread IDs or message IDs instead of only sender address if multiple campaigns may target the same person.

**Sub-workflow setup:**  
This workflow does **not** use any sub-workflows or Execute Workflow nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Book a Free Consulting Call | https://smoothwork.ai/book-a-call/ |
| SmoothWork contact email: hello@smoothwork.ai | Creator contact |
| Check us on YouTube | https://www.youtube.com/@vasarmilan |
| Add me on Linkedin | https://www.linkedin.com/in/mil%C3%A1n-v%C3%A1s%C3%A1rhelyi-3a9985123/ |
| Video Walkthrough | https://www.youtube.com/watch?v=jxT6XO4eUwI |
| Workflow note: create columns `To`, `Subject`, `Message`, `Status`, and `Response` before use | Setup context |
| Workflow note: update document ID and sheet references to your own spreadsheet | Setup context |
| Workflow note: outbound filter expects the label `To send` in the `Status` column | Configuration context |