Filter and label Gmail emails using keyword-based rules and Google Sheets logging

https://n8nworkflows.xyz/workflows/filter-and-label-gmail-emails-using-keyword-based-rules-and-google-sheets-logging-13903


# Filter and label Gmail emails using keyword-based rules and Google Sheets logging

## 1. Workflow Overview

This workflow monitors a Gmail inbox, fetches each newly received email in full, classifies it using keyword/domain-based rules, then applies Gmail labels and optionally logs the result to Google Sheets.

Its main use case is automated inbox triage:
- detect likely cold outreach and suppress it,
- detect marketing/promotional emails,
- flag ambiguous messages for manual review,
- preserve legitimate operational/business emails.

The workflow contains one entry point and then splits into classification-driven branches.

### 1.1 Input Reception and Message Retrieval
The workflow starts when Gmail receives a new message. It then retrieves the full message content so the later logic can access sender, subject, and body text.

### 1.2 Email Data Preparation
The workflow normalizes key fields into a simpler structure: combined text content, sender details, message ID, and subject.

### 1.3 Rule-Based Classification
A Code node performs all classification logic. It uses:
- trusted sender domains,
- legitimate business indicators,
- cold outreach keywords,
- sales-tool sender domains,
- suspicious “needs review” phrasing,
- unsubscribe markers.

This node returns a normalized classification payload.

### 1.4 Cold Outreach Handling
If the email is classified as `cold_outreach`, the workflow:
- adds a Gmail label,
- removes the message from Inbox,
- marks it as read,
- prepares a structured logging row,
- appends it to Google Sheets.

### 1.5 Needs Review Handling
If the email is classified as `needs_review`, the workflow:
- adds a Gmail label,
- prepares structured logging fields,
- appends the case to Google Sheets.

### 1.6 Marketing Handling
If the email is classified as `marketing`, the workflow:
- adds a Gmail label,
- prepares structured logging fields,
- appends the result to Google Sheets.

### 1.7 Legitimate Handling
If the email is neither cold outreach, nor needs review, nor marketing, it is treated as `legitimate`. The workflow:
- adds a Gmail label,
- prepares structured logging fields,
- appends the result to Google Sheets.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Message Retrieval

**Overview:**  
This block listens for incoming Gmail messages and retrieves the full email object. It converts the lightweight trigger event into a richer payload for downstream analysis.

**Nodes Involved:**  
- Gmail Trigger
- Get a message

### Node: Gmail Trigger
- **Type and technical role:** `n8n-nodes-base.gmailTrigger` — polling trigger for new Gmail messages.
- **Configuration choices:**  
  - Polling mode is set to run every minute.
  - No additional Gmail filters are configured.
- **Key expressions or variables used:**  
  - None.
- **Input and output connections:**  
  - Entry point of the workflow.
  - Outputs to `Get a message`.
- **Version-specific requirements:**  
  - Type version `1.2`.
  - Requires Gmail trigger support in the installed n8n version.
- **Edge cases or potential failure types:**  
  - Gmail authentication failure.
  - Polling delay or API quota issues.
  - Trigger may pick up emails you did not intend if filters remain empty.
- **Sub-workflow reference:**  
  - None.

### Node: Get a message
- **Type and technical role:** `n8n-nodes-base.gmail` — fetches the full Gmail message by ID.
- **Configuration choices:**  
  - Operation: `get`
  - `messageId` is pulled from the trigger event: `{{ $json.id }}`
  - `simple` is disabled, so the node returns a fuller Gmail message structure.
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Gmail Trigger`
  - Output to `Extract Email Data`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - If the trigger payload lacks `id`, the lookup fails.
  - Gmail API permissions may allow trigger events but not full message retrieval if scopes are incomplete.
  - Message may become unavailable between trigger and retrieval in rare race conditions.
- **Sub-workflow reference:**  
  - None.

---

## 2.2 Email Data Preparation

**Overview:**  
This block extracts and reshapes the Gmail message into fields that are easier to reference. It also creates a combined content field used by the classifier.

**Nodes Involved:**  
- Extract Email Data

### Node: Extract Email Data
- **Type and technical role:** `n8n-nodes-base.set` — creates a simplified analysis payload.
- **Configuration choices:**  
  - Assigns:
    - `email_content` = subject + text body
    - `sender_email` = first sender address
    - `sender_name` = first sender name or `Unknown`
    - `message_id` = Gmail message ID
    - `Subject` = subject
  - `executeOnce` is enabled.
  - `alwaysOutputData` is enabled.
- **Key expressions or variables used:**  
  - `={{ $json.subject + ' ' + $json.text }}`
  - `={{ $json.from.value[0].address }}`
  - `={{ $json.from.value[0].name || 'Unknown' }}`
  - `={{ $json.id }}`
  - `={{ $json.subject }}`
- **Input and output connections:**  
  - Input from `Get a message`
  - Output to `Analyze Email Content`
- **Version-specific requirements:**  
  - Type version `3.3`
- **Edge cases or potential failure types:**  
  - If `subject` or `text` is missing, expression output may become incomplete or fail depending on payload shape.
  - `from.value[0]` assumes sender parsing succeeded; malformed or unusual email formats can break this expression.
  - `executeOnce` can be problematic if later adapted for batched processing.
- **Sub-workflow reference:**  
  - None.

---

## 2.3 Rule-Based Classification

**Overview:**  
This is the core decision engine. It scores the email and assigns one of four categories: `cold_outreach`, `marketing`, `needs_review`, or `legitimate`.

**Nodes Involved:**  
- Analyze Email Content
- Check If Cold Outreach
- Check if Needs_review
- Check if Marketing

### Node: Analyze Email Content
- **Type and technical role:** `n8n-nodes-base.code` — custom JavaScript classifier.
- **Configuration choices:**  
  - Defines internal arrays for:
    - trusted domains,
    - legitimate patterns,
    - cold outreach keywords,
    - sales tool domains,
    - review indicators.
  - Converts content and sender email to lowercase for case-insensitive checks.
  - Extracts sender domain from email.
  - Prioritizes trusted domain classification first.
  - Calculates:
    - `legitimate_score`
    - `review_score`
    - detected keywords
    - sales-tool detection
    - unsubscribe flag
  - Returns a structured classification object.
- **Key expressions or variables used:**  
  - Reads from input:
    - `$input.first().json.email_content`
    - `$input.first().json.sender_email`
    - `$input.first().json.sender_name`
    - `$input.first().json.message_id`
  - Returns fields such as:
    - `email_type`
    - `confidence`
    - `classification_reason`
    - `found_keywords`
    - `found_legitimate_patterns`
    - `found_review_indicators`
    - `analysis_timestamp`
- **Input and output connections:**  
  - Input from `Extract Email Data`
  - Output to `Check If Cold Outreach`
- **Version-specific requirements:**  
  - Type version `2`
  - Requires Code node JavaScript execution support.
- **Edge cases or potential failure types:**  
  - If `sender_email` is missing, domain extraction becomes unreliable.
  - Name repetition check uses the sender name instead of the recipient name; this may not capture the intended personalization signal.
  - The code assumes textual body content exists and is readable.
  - Classification defaults to `legitimate` for unknown content, which may be too permissive for some use cases.
  - Trusted-domain matching uses `includes()`, which can create false positives for deceptive domains containing a trusted string.
- **Sub-workflow reference:**  
  - None.

### Node: Check If Cold Outreach
- **Type and technical role:** `n8n-nodes-base.if` — branch on `email_type`.
- **Configuration choices:**  
  - Condition: `email_type` equals `cold_outreach`
  - True output goes to outreach handling.
  - False output continues to next check.
- **Key expressions or variables used:**  
  - `={{ $json.email_type }}`
- **Input and output connections:**  
  - Input from `Analyze Email Content`
  - True output to `Label as Outreach`
  - False output to `Check if Needs_review`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - If `email_type` is absent or misspelled, the message falls through the false branch.
- **Sub-workflow reference:**  
  - None.

### Node: Check if Needs_review
- **Type and technical role:** `n8n-nodes-base.if` — branch on `needs_review`.
- **Configuration choices:**  
  - Condition: `email_type` equals `needs_review`
- **Key expressions or variables used:**  
  - `={{ $json.email_type }}`
- **Input and output connections:**  
  - Input from `Check If Cold Outreach` false branch
  - True output to `Label as Needs Review`
  - False output to `Check if Marketing`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Same as above: absent or malformed `email_type` causes fallback to later logic.
- **Sub-workflow reference:**  
  - None.

### Node: Check if Marketing
- **Type and technical role:** `n8n-nodes-base.if` — final classification split before defaulting to legitimate.
- **Configuration choices:**  
  - Condition: `email_type` equals `marketing`
  - True output goes to marketing handling.
  - False output goes to legitimate handling.
- **Key expressions or variables used:**  
  - `={{ $json.email_type }}`
- **Input and output connections:**  
  - Input from `Check if Needs_review` false branch
  - True output to `Label as Marketing/Ads`
  - False output to `Label as Legitimate`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Any unknown classification value is implicitly treated as legitimate because the false branch leads there.
- **Sub-workflow reference:**  
  - None.

---

## 2.4 Cold Outreach Handling

**Overview:**  
This block performs the strongest cleanup actions. Cold outreach messages are labeled, removed from Inbox, marked read, and logged.

**Nodes Involved:**  
- Label as Outreach
- Remove from Inbox
- Mark as Read
- Edit Fields (outreach)
- Log to Spreadsheet (outreach)

### Node: Label as Outreach
- **Type and technical role:** `n8n-nodes-base.gmail` — adds a Gmail label to the message.
- **Configuration choices:**  
  - Operation: `addLabels`
  - Label ID: `Label_1684442270786871755`
  - Message ID from classifier output.
- **Key expressions or variables used:**  
  - `={{ $json.message_id }}`
- **Input and output connections:**  
  - Input from `Check If Cold Outreach` true branch
  - Output to `Remove from Inbox`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Label ID must exist in the connected Gmail account.
  - Imported workflows often fail here until the user replaces label IDs with real IDs from their mailbox.
- **Sub-workflow reference:**  
  - None.

### Node: Remove from Inbox
- **Type and technical role:** `n8n-nodes-base.gmail` — removes the `INBOX` label.
- **Configuration choices:**  
  - Operation: `removeLabels`
  - Label removed: `INBOX`
  - Message ID: `={{ $json.id }}`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Label as Outreach`
  - Output to `Mark as Read`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - This node uses `id`, not `message_id`. It depends on the Gmail node output preserving a field named `id`.
  - If Gmail output shape differs, the node can fail or target no message.
- **Sub-workflow reference:**  
  - None.

### Node: Mark as Read
- **Type and technical role:** `n8n-nodes-base.gmail` — marks the message as read.
- **Configuration choices:**  
  - Operation: `markAsRead`
  - Message ID: `={{ $json.id }}`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input from `Remove from Inbox`
  - Output to `Edit Fields (outreach)`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Same output-shape dependency as `Remove from Inbox`.
- **Sub-workflow reference:**  
  - None.

### Node: Edit Fields (outreach)
- **Type and technical role:** `n8n-nodes-base.set` — creates a clean row for spreadsheet logging.
- **Configuration choices:**  
  - Pulls values from previous nodes via direct node references, not only current input.
  - Produces:
    - `analysis_timestamp`
    - `message_id`
    - `sender_email`
    - `sender_name`
    - `subject`
    - `email_type`
    - `confidence`
    - `keywords_found`
- **Key expressions or variables used:**  
  - `{{ $('Analyze Email Content').first().json.analysis_timestamp }}`
  - `{{ $('Analyze Email Content').first().json.message_id }}`
  - `{{ $('Analyze Email Content').first().json.sender_email }}`
  - `{{ $('Extract Email Data').first().json.sender_name }}`
  - `{{ $('Extract Email Data').first().json.Subject }}`
  - `{{ $('Analyze Email Content').first().json.found_keywords.join(', ') }}`
- **Input and output connections:**  
  - Input from `Mark as Read`
  - Output to `Log to Spreadsheet (outreach)`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - If the referenced nodes do not execute in the current run, expressions can fail.
  - If `found_keywords` is not an array, `.join(', ')` fails.
- **Sub-workflow reference:**  
  - None.

### Node: Log to Spreadsheet (outreach)
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends a log row.
- **Configuration choices:**  
  - Operation: `append`
  - Mapped columns:
    - timestamp
    - message_id
    - sender_email
    - sender_name
    - subject
    - email_type
    - confidence
    - keywords_found
  - Sheet name points to `Sheet1` (`gid=0`)
  - `documentId` is currently empty and must be supplied.
- **Key expressions or variables used:**  
  - Standard field mappings from current JSON.
- **Input and output connections:**  
  - Input from `Edit Fields (outreach)`
  - No downstream connection.
- **Version-specific requirements:**  
  - Type version `4`
- **Edge cases or potential failure types:**  
  - Missing Google Sheets credentials.
  - Empty `documentId` means this node will not work until configured.
  - Sheet schema must contain matching headers if using mapped columns.
- **Sub-workflow reference:**  
  - None.

---

## 2.5 Needs Review Handling

**Overview:**  
This block labels uncertain emails for manual inspection and logs them. It does not remove them from the Inbox or mark them read.

**Nodes Involved:**  
- Label as Needs Review
- Edit Fields (needs review)
- Log to Spreadsheet (needs review)

### Node: Label as Needs Review
- **Type and technical role:** `n8n-nodes-base.gmail` — adds the review label.
- **Configuration choices:**  
  - Operation: `addLabels`
  - Label ID: `Label_221566828550691632`
  - Message ID from classifier output.
- **Key expressions or variables used:**  
  - `={{ $json.message_id }}`
- **Input and output connections:**  
  - Input from `Check if Needs_review` true branch
  - Output to `Edit Fields (needs review)`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Invalid label ID after import is the most common issue.
- **Sub-workflow reference:**  
  - None.

### Node: Edit Fields (needs review)
- **Type and technical role:** `n8n-nodes-base.set` — prepares spreadsheet row data.
- **Configuration choices:**  
  - Same field structure as outreach logging.
- **Key expressions or variables used:**  
  - Same pattern as outreach edit node.
- **Input and output connections:**  
  - Input from `Label as Needs Review`
  - Output to `Log to Spreadsheet (needs review)`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - Same node-reference dependency risks as other Set logging nodes.
- **Sub-workflow reference:**  
  - None.

### Node: Log to Spreadsheet (needs review)
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends a review log row.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet points to tab `Needs_review`
  - `documentId` is empty and must be configured.
- **Key expressions or variables used:**  
  - Standard mapped current fields.
- **Input and output connections:**  
  - Input from `Edit Fields (needs review)`
  - No downstream connection.
- **Version-specific requirements:**  
  - Type version `4`
- **Edge cases or potential failure types:**  
  - Empty `documentId`
  - Missing sheet or column mismatch
  - Credential errors
- **Sub-workflow reference:**  
  - None.

---

## 2.6 Marketing Handling

**Overview:**  
This block applies a marketing/promotions label and writes the result to Google Sheets. It is reached only if the message was not cold outreach and not needs review.

**Nodes Involved:**  
- Label as Marketing/Ads
- Edit Fields (marketing)
- Log to Spreadsheet (marketing/Ads)

### Node: Label as Marketing/Ads
- **Type and technical role:** `n8n-nodes-base.gmail` — adds the marketing label.
- **Configuration choices:**  
  - Operation: `addLabels`
  - Label ID: `Label_999978575131975706`
- **Key expressions or variables used:**  
  - `={{ $json.message_id }}`
- **Input and output connections:**  
  - Input from `Check if Marketing` true branch
  - Output to `Edit Fields (marketing)`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - Label ID must be replaced if moving to another Gmail account.
- **Sub-workflow reference:**  
  - None.

### Node: Edit Fields (marketing)
- **Type and technical role:** `n8n-nodes-base.set` — prepares structured row data.
- **Configuration choices:**  
  - Same logging fields as the other category branches.
- **Key expressions or variables used:**  
  - Same node reference expressions as the other edit nodes.
- **Input and output connections:**  
  - Input from `Label as Marketing/Ads`
  - Output to `Log to Spreadsheet (marketing/Ads)`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - Same expression dependency issues as the parallel edit nodes.
- **Sub-workflow reference:**  
  - None.

### Node: Log to Spreadsheet (marketing/Ads)
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends marketing log rows.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet tab points to `Marketing`
  - `documentId` is empty and must be filled in.
- **Key expressions or variables used:**  
  - Standard mapped fields.
- **Input and output connections:**  
  - Input from `Edit Fields (marketing)`
  - No downstream connection.
- **Version-specific requirements:**  
  - Type version `4`
- **Edge cases or potential failure types:**  
  - Missing Google Sheets spreadsheet ID
  - Tab/header mismatch
- **Sub-workflow reference:**  
  - None.

---

## 2.7 Legitimate Handling

**Overview:**  
This is the default branch for messages considered safe or operational. The workflow labels them and logs the result.

**Nodes Involved:**  
- Label as Legitimate
- Edit Fields (Legitimate)
- Log to Spreadsheet (legitimate)

### Node: Label as Legitimate
- **Type and technical role:** `n8n-nodes-base.gmail` — adds the legitimate label.
- **Configuration choices:**  
  - Operation: `addLabels`
  - Label ID: `Label_9187111232+1234567890`
- **Key expressions or variables used:**  
  - `={{ $json.message_id }}`
- **Input and output connections:**  
  - Input from `Check if Marketing` false branch
  - Output to `Edit Fields (Legitimate)`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - The label ID format looks unusual and should be verified in Gmail before use.
- **Sub-workflow reference:**  
  - None.

### Node: Edit Fields (Legitimate)
- **Type and technical role:** `n8n-nodes-base.set` — prepares spreadsheet row data for legitimate messages.
- **Configuration choices:**  
  - Same field structure as all other logging branches.
- **Key expressions or variables used:**  
  - Same node references as the other edit nodes.
- **Input and output connections:**  
  - Input from `Label as Legitimate`
  - Output to `Log to Spreadsheet (legitimate)`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - Same expression dependency issues as other edit nodes.
- **Sub-workflow reference:**  
  - None.

### Node: Log to Spreadsheet (legitimate)
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends log rows for legitimate emails.
- **Configuration choices:**  
  - Operation: `append`
  - Sheet tab points to `Legitimate`
  - `documentId` is empty and must be set.
- **Key expressions or variables used:**  
  - Standard mapped fields.
- **Input and output connections:**  
  - Input from `Edit Fields (Legitimate)`
  - No downstream connection.
- **Version-specific requirements:**  
  - Type version `4`
- **Edge cases or potential failure types:**  
  - Same Sheets setup issues as other log nodes.
- **Sub-workflow reference:**  
  - None.

---

## 2.8 Sticky Notes / Embedded Documentation

**Overview:**  
These nodes are documentation-only elements inside the canvas. They provide setup guidance, logic explanation, and logging notes.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Configuration choices:**  
  - Describes overall workflow and Gmail label setup.
- **Key expressions or variables used:**  
  - None.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Type version `1`
- **Edge cases or potential failure types:**  
  - None; informational only.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Configuration choices:**  
  - Explains extraction and analysis area.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Type version `1`
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Configuration choices:**  
  - Explains keyword filtering and customization.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Type version `1`
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Configuration choices:**  
  - Explains post-classification actions and optional Google Sheets logging.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Type version `1`
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | Gmail Trigger | Entry point; detects new Gmail messages |  | Get a message | ## Workflow Overview +Gmail Setup<br>This workflow automatically filters incoming Gmail emails using keyword detection.<br>The workflow analyzes the email subject and snippet to determine whether the email is likely:<br>• Cold outreach<br>• Marketing / promotional<br>• Needs manual review<br>• Legitimate communication<br>Before running the workflow, create the following Gmail labels:<br>Cold Outreach<br>Marketing<br>Needs Review<br>Legitimate<br>You can create these labels in Gmail by selecting "Create new label" in the left sidebar.<br>The Gmail trigger node listens for new incoming emails and starts the workflow whenever a message arrives. |
| Get a message | Gmail | Retrieves full Gmail message details | Gmail Trigger | Extract Email Data | ## Email Data Extraction and Analyze<br>These nodes retrieve the full Gmail message and analyze them.<br>The workflow extracts useful fields such as:<br>• Sender email<br>• Subject line<br>• Email snippet<br>• Timestamp<br>These values are then used in the filtering logic to determine the email category. |
| Extract Email Data | Set | Normalizes sender, subject, body, and message ID | Get a message | Analyze Email Content | ## Email Data Extraction and Analyze<br>These nodes retrieve the full Gmail message and analyze them.<br>The workflow extracts useful fields such as:<br>• Sender email<br>• Subject line<br>• Email snippet<br>• Timestamp<br>These values are then used in the filtering logic to determine the email category. |
| Analyze Email Content | Code | Applies rule-based classification logic | Extract Email Data | Check If Cold Outreach | ## Email Data Extraction and Analyze<br>These nodes retrieve the full Gmail message and analyze them.<br>The workflow extracts useful fields such as:<br>• Sender email<br>• Subject line<br>• Email snippet<br>• Timestamp<br>These values are then used in the filtering logic to determine the email category. |
| Check If Cold Outreach | If | Routes cold outreach vs other email types | Analyze Email Content | Label as Outreach; Check if Needs_review | ## Keyword Filtering Logic<br>This section checks the email subject and snippet for keywords commonly associated with:<br>Cold outreach:<br>sales outreach, partnership opportunity, quick call, growth strategy, etc.<br>Marketing emails:<br>newsletter, promotion, discount, marketing campaign, etc.<br>If the workflow detects these patterns, the email is classified accordingly.<br>Emails that do not match clear patterns are sent to the "Needs Review" category.<br>You can customize these keyword lists to better match your inbox. |
| Check if Needs_review | If | Routes needs-review emails vs next classification step | Check If Cold Outreach | Label as Needs Review; Check if Marketing | ## Keyword Filtering Logic<br>This section checks the email subject and snippet for keywords commonly associated with:<br>Cold outreach:<br>sales outreach, partnership opportunity, quick call, growth strategy, etc.<br>Marketing emails:<br>newsletter, promotion, discount, marketing campaign, etc.<br>If the workflow detects these patterns, the email is classified accordingly.<br>Emails that do not match clear patterns are sent to the "Needs Review" category.<br>You can customize these keyword lists to better match your inbox. |
| Check if Marketing | If | Routes marketing emails vs legitimate default | Check if Needs_review | Label as Marketing/Ads; Label as Legitimate | ## Keyword Filtering Logic<br>This section checks the email subject and snippet for keywords commonly associated with:<br>Cold outreach:<br>sales outreach, partnership opportunity, quick call, growth strategy, etc.<br>Marketing emails:<br>newsletter, promotion, discount, marketing campaign, etc.<br>If the workflow detects these patterns, the email is classified accordingly.<br>Emails that do not match clear patterns are sent to the "Needs Review" category.<br>You can customize these keyword lists to better match your inbox. |
| Label as Outreach | Gmail | Adds Gmail label for cold outreach | Check If Cold Outreach | Remove from Inbox | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Remove from Inbox | Gmail | Removes Inbox label from outreach emails | Label as Outreach | Mark as Read | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Mark as Read | Gmail | Marks outreach emails as read | Remove from Inbox | Edit Fields (outreach) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Edit Fields (outreach) | Set | Prepares outreach log row | Mark as Read | Log to Spreadsheet (outreach) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Log to Spreadsheet (outreach) | Google Sheets | Logs outreach classification | Edit Fields (outreach) |  | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Label as Needs Review | Gmail | Adds Gmail label for review cases | Check if Needs_review | Edit Fields (needs review) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Edit Fields (needs review) | Set | Prepares needs-review log row | Label as Needs Review | Log to Spreadsheet (needs review) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Log to Spreadsheet (needs review) | Google Sheets | Logs needs-review classification | Edit Fields (needs review) |  | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Label as Marketing/Ads | Gmail | Adds Gmail label for marketing emails | Check if Marketing | Edit Fields (marketing) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Edit Fields (marketing) | Set | Prepares marketing log row | Label as Marketing/Ads | Log to Spreadsheet (marketing/Ads) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Log to Spreadsheet (marketing/Ads) | Google Sheets | Logs marketing classification | Edit Fields (marketing) |  | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Label as Legitimate | Gmail | Adds Gmail label for legitimate emails | Check if Marketing | Edit Fields (Legitimate) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Edit Fields (Legitimate) | Set | Prepares legitimate log row | Label as Legitimate | Log to Spreadsheet (legitimate) | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Log to Spreadsheet (legitimate) | Google Sheets | Logs legitimate classification | Edit Fields (Legitimate) |  | ##Email Actions and Optional Logging<br>After classification, the workflow applies the appropriate Gmail label.<br>Actions performed:<br>• Apply the label (Cold Outreach / Marketing / Needs Review / Legitimate)<br>• Mark the email as read<br>• Optionally remove it from the inbox<br>Optional: Google Sheets logging<br>The workflow can log processed emails to Google Sheets to keep a record of:<br>• timestamp<br>• sender<br>• subject<br>• classification<br>Note: make sure to make a google sheet in advance with those columns.<br>You can disable the logging node if you do not need this feature. |
| Sticky Note | Sticky Note | Canvas documentation for setup |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for extraction/analysis |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for keyword logic |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for actions/logging |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create required Gmail labels first**
   - In Gmail, manually create these labels:
     - Cold Outreach
     - Marketing
     - Needs Review
     - Legitimate
   - After creating them, note their Gmail label IDs if needed in n8n. In imported workflows, label IDs are account-specific and must usually be reselected.

2. **Optionally prepare Google Sheets for logging**
   - Create a spreadsheet with separate tabs, for example:
     - `Sheet1` for outreach
     - `Needs_review`
     - `Marketing`
     - `Legitimate`
   - Add these columns to each tab:
     - `timestamp`
     - `message_id`
     - `sender_email`
     - `sender_name`
     - `subject`
     - `email_type`
     - `confidence`
     - `keywords_found`

3. **Create a new workflow**
   - Name it something like: `Smart Email Filtering System (using keywords)`.

4. **Add a Gmail Trigger node**
   - Type: `Gmail Trigger`
   - Configure Gmail credentials.
   - Set polling to every minute.
   - Leave filters empty unless you want only certain emails to trigger the workflow.
   - This is the entry point.

5. **Add a Gmail node named `Get a message`**
   - Type: `Gmail`
   - Operation: `Get`
   - Set `Message ID` to `{{ $json.id }}`
   - Disable simplified output (`simple: false`) so full message metadata is available.
   - Connect `Gmail Trigger -> Get a message`.

6. **Add a Set node named `Extract Email Data`**
   - Create these fields:
     - `email_content` = `{{ $json.subject + ' ' + $json.text }}`
     - `sender_email` = `{{ $json.from.value[0].address }}`
     - `sender_name` = `{{ $json.from.value[0].name || 'Unknown' }}`
     - `message_id` = `{{ $json.id }}`
     - `Subject` = `{{ $json.subject }}`
   - Enable `Always Output Data`.
   - Keep `Execute Once` enabled only if processing one message per execution is acceptable.
   - Connect `Get a message -> Extract Email Data`.

7. **Add a Code node named `Analyze Email Content`**
   - Paste the JavaScript classification logic from the workflow.
   - Ensure it returns a single JSON object with fields including:
     - `message_id`
     - `email_type`
     - `confidence`
     - `classification_reason`
     - `found_keywords`
     - `found_legitimate_patterns`
     - `found_review_indicators`
     - `sales_tool_detected`
     - `has_unsubscribe`
     - `sender_email`
     - `sender_domain`
     - `legitimate_score`
     - `review_score`
     - `analysis_timestamp`
   - Connect `Extract Email Data -> Analyze Email Content`.

8. **Add an If node named `Check If Cold Outreach`**
   - Condition:
     - left value: `{{ $json.email_type }}`
     - operator: `equals`
     - right value: `cold_outreach`
   - Connect `Analyze Email Content -> Check If Cold Outreach`.

9. **Create cold outreach Gmail action node**
   - Add a Gmail node named `Label as Outreach`
   - Operation: `Add Labels`
   - Select the Gmail label corresponding to `Cold Outreach`
   - `Message ID` = `{{ $json.message_id }}`
   - Connect the **true** output of `Check If Cold Outreach` to this node.

10. **Add node `Remove from Inbox`**
    - Type: `Gmail`
    - Operation: `Remove Labels`
    - Label to remove: `INBOX`
    - `Message ID` = `{{ $json.id }}`
    - Connect `Label as Outreach -> Remove from Inbox`.
    - Recommended improvement: if your Gmail node output does not preserve `id`, use `message_id` consistently instead.

11. **Add node `Mark as Read`**
    - Type: `Gmail`
    - Operation: `Mark as Read`
    - `Message ID` = `{{ $json.id }}`
    - Connect `Remove from Inbox -> Mark as Read`.

12. **Add logging preparation node for outreach**
    - Type: `Set`
    - Name: `Edit Fields (outreach)`
    - Add:
      - `analysis_timestamp` = `{{ $('Analyze Email Content').first().json.analysis_timestamp }}`
      - `message_id` = `{{ $('Analyze Email Content').first().json.message_id }}`
      - `sender_email` = `{{ $('Analyze Email Content').first().json.sender_email }}`
      - `sender_name` = `{{ $('Extract Email Data').first().json.sender_name }}`
      - `subject` = `{{ $('Extract Email Data').first().json.Subject }}`
      - `email_type` = `{{ $('Analyze Email Content').first().json.email_type }}`
      - `confidence` = `{{ $('Analyze Email Content').first().json.confidence }}`
      - `keywords_found` = `{{ $('Analyze Email Content').first().json.found_keywords.join(', ') }}`
    - Connect `Mark as Read -> Edit Fields (outreach)`.

13. **Add outreach Google Sheets logging node**
    - Type: `Google Sheets`
    - Name: `Log to Spreadsheet (outreach)`
    - Operation: `Append`
    - Configure Google Sheets credentials.
    - Select spreadsheet document ID.
    - Select tab `Sheet1` or your outreach tab.
    - Map the 8 columns from the Set node.
    - Connect `Edit Fields (outreach) -> Log to Spreadsheet (outreach)`.

14. **Add an If node named `Check if Needs_review`**
    - Connect the **false** output of `Check If Cold Outreach` to this node.
    - Condition:
      - `{{ $json.email_type }}`
      - equals
      - `needs_review`

15. **Create needs-review labeling node**
    - Add Gmail node `Label as Needs Review`
    - Operation: `Add Labels`
    - Select the Gmail label corresponding to `Needs Review`
    - `Message ID` = `{{ $json.message_id }}`
    - Connect the **true** output of `Check if Needs_review` to it.

16. **Add needs-review logging preparation node**
    - Add Set node `Edit Fields (needs review)`
    - Use the same fields and expressions as outreach.
    - Connect `Label as Needs Review -> Edit Fields (needs review)`.

17. **Add needs-review Sheets node**
    - Add Google Sheets node `Log to Spreadsheet (needs review)`
    - Operation: `Append`
    - Select spreadsheet and tab `Needs_review`
    - Map the same 8 columns.
    - Connect `Edit Fields (needs review) -> Log to Spreadsheet (needs review)`.

18. **Add an If node named `Check if Marketing`**
    - Connect the **false** output of `Check if Needs_review` to this node.
    - Condition:
      - `{{ $json.email_type }}`
      - equals
      - `marketing`

19. **Create marketing labeling node**
    - Add Gmail node `Label as Marketing/Ads`
    - Operation: `Add Labels`
    - Select the Gmail label corresponding to `Marketing`
    - `Message ID` = `{{ $json.message_id }}`
    - Connect the **true** output of `Check if Marketing` to it.

20. **Add marketing logging preparation node**
    - Add Set node `Edit Fields (marketing)`
    - Use the same field mappings as the previous Set logging nodes.
    - Connect `Label as Marketing/Ads -> Edit Fields (marketing)`.

21. **Add marketing Sheets node**
    - Add Google Sheets node `Log to Spreadsheet (marketing/Ads)`
    - Operation: `Append`
    - Select spreadsheet and tab `Marketing`
    - Map the same 8 fields.
    - Connect `Edit Fields (marketing) -> Log to Spreadsheet (marketing/Ads)`.

22. **Create legitimate labeling node**
    - Add Gmail node `Label as Legitimate`
    - Operation: `Add Labels`
    - Select the Gmail label corresponding to `Legitimate`
    - `Message ID` = `{{ $json.message_id }}`
    - Connect the **false** output of `Check if Marketing` to it.

23. **Add legitimate logging preparation node**
    - Add Set node `Edit Fields (Legitimate)`
    - Use the same field mappings as the other Set nodes.
    - Connect `Label as Legitimate -> Edit Fields (Legitimate)`.

24. **Add legitimate Sheets node**
    - Add Google Sheets node `Log to Spreadsheet (legitimate)`
    - Operation: `Append`
    - Select spreadsheet and tab `Legitimate`
    - Map the same 8 fields.
    - Connect `Edit Fields (Legitimate) -> Log to Spreadsheet (legitimate)`.

25. **Configure credentials**
    - **Gmail credentials:** required for:
      - Gmail Trigger
      - Get a message
      - all Gmail action nodes
    - **Google Sheets credentials:** required for:
      - all four Google Sheets nodes
    - Ensure the Gmail connection has enough permission to:
      - read messages,
      - modify labels,
      - mark as read,
      - remove Inbox label.

26. **Validate label IDs and spreadsheet IDs**
    - Re-select each Gmail label in the Gmail nodes after import/manual build.
    - Fill every Google Sheets `documentId`; the source workflow leaves them blank.
    - Confirm each sheet tab exists.

27. **Test each branch**
    - Send or use sample emails that should match:
      - cold outreach,
      - marketing,
      - needs review,
      - legitimate.
    - Verify:
      - label applied correctly,
      - Inbox removal only happens for outreach,
      - mark-as-read only happens for outreach,
      - sheet log row is appended correctly.

28. **Activate the workflow**
    - Once all credentials, label mappings, and spreadsheet references are valid, activate the workflow.

### Sub-workflow setup
- This workflow does **not** invoke any sub-workflows.
- There are no Execute Workflow nodes or workflow-call dependencies.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is inactive in the provided export. | Activate only after credentials, labels, and Sheets targets are configured. |
| Gmail label IDs are mailbox-specific and usually cannot be reused as-is across accounts. | Re-select labels manually in each Gmail node. |
| All Google Sheets logging nodes have an empty `documentId` in the provided workflow. | Must be filled before logging works. |
| The classifier defaults unknown messages to `legitimate`. | Consider changing this to `needs_review` if you prefer a safer posture. |
| Trusted-domain matching uses substring inclusion. | Tighten this to exact-domain or subdomain-safe matching if spoof resistance matters. |
| The sticky notes instruct users to create labels in Gmail before building. | Gmail setup guidance embedded in canvas notes. |
| Logging is optional according to the workflow notes. | You can disable or delete all Google Sheets nodes if not needed. |