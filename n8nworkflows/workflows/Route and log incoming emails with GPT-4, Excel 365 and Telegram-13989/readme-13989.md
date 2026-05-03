Route and log incoming emails with GPT-4, Excel 365 and Telegram

https://n8nworkflows.xyz/workflows/route-and-log-incoming-emails-with-gpt-4--excel-365-and-telegram-13989


# Route and log incoming emails with GPT-4, Excel 365 and Telegram

# Route and log incoming emails with GPT-4, Excel 365 and Telegram

## 1. Workflow Overview

This workflow monitors an IMAP inbox, extracts key information from each incoming email, sends the email content to an OpenAI model for categorization or summarization, parses the AI output, logs the result into an Excel 365 table, checks whether the detected department exists in a reference table, and then sends follow-up notifications through Telegram and email.

Its main use cases are:

- Automated triage of incoming emails
- AI-assisted classification of messages into departments or categories
- Structured logging of email activity into Microsoft Excel 365
- Internal alerting when a department match is found or validated
- Optional downstream routing by email

Because the JSON contains mostly empty parameter objects, the workflow structure is clear but many node-level business settings are not preserved in the exported data shown here. This means the logic can be reconstructed reliably, but some exact field mappings, prompts, filters, table names, and message templates must be redefined manually.

### 1.1 Input Reception
The workflow starts when a new email is received through an IMAP trigger, then normalizes the incoming data into a cleaner field set.

### 1.2 AI Interpretation
The extracted email data is passed to an OpenAI chat/model node, which likely produces a structured summary or classification. A Code node then parses that model output.

### 1.3 Logging Preparation and Persistence
The parsed AI result is converted into an Excel-friendly row structure and appended to a Microsoft Excel 365 table.

### 1.4 Department Validation and Routing
The prepared row is also used to perform a department lookup in Excel. The result is evaluated with an IF node, and if the validation path succeeds, the workflow sends a Telegram notification and an email.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Input Reception

### Overview
This block ingests incoming email messages from an IMAP mailbox and prepares the fields needed by downstream AI processing. It is the workflow’s only explicit entry point.

### Nodes Involved
- Email Trigger (IMAP)
- Extract Email Fields

### Node Details

#### Email Trigger (IMAP)
- **Type and technical role:** `n8n-nodes-base.emailReadImap`  
  Trigger node that starts the workflow when new email messages are detected in an IMAP mailbox.
- **Configuration choices:**  
  The exported JSON shows no explicit parameters, so mailbox selection, polling behavior, post-processing rules, and message filters are not visible here. These must be configured manually.
- **Key expressions or variables used:**  
  Not shown in the JSON. Typical outputs from this node include sender, recipient, subject, text body, HTML body, date, and attachments metadata.
- **Input and output connections:**  
  - Input: none, this is the workflow trigger
  - Output: connected to **Extract Email Fields**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`. IMAP credential handling and parameter layout should match the Email IMAP Trigger node version available in the installed n8n release.
- **Edge cases or potential failure types:**  
  - IMAP authentication failure
  - Wrong host/port or TLS settings
  - Mailbox/folder not found
  - Duplicate reads depending on trigger settings
  - Large emails or malformed MIME structure
  - Attachment-heavy messages causing payload bloat
- **Sub-workflow reference:**  
  None

#### Extract Email Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  A Set node used to normalize, rename, or select only the relevant email fields for AI processing.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the exact mapped fields are not visible. In practice, this node likely selects fields such as:
  - sender email
  - sender name
  - subject
  - body text
  - received date
  - message ID
- **Key expressions or variables used:**  
  Not shown. Typical examples would be expressions referencing incoming IMAP fields such as subject/body/from.
- **Input and output connections:**  
  - Input: **Email Trigger (IMAP)**
  - Output: **Message a model**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`. Field behavior depends on the Set node version in n8n, especially whether it keeps only set fields or also passes through existing JSON.
- **Edge cases or potential failure types:**  
  - Missing body text or subject
  - Unexpected input shape from IMAP
  - Expression errors if field names differ from expected
  - HTML-only emails requiring conversion to plain text first
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: AI Interpretation

### Overview
This block sends the extracted email content to an OpenAI model, then parses the AI response into structured fields that the rest of the workflow can use. It is the core intelligence layer of the process.

### Nodes Involved
- Message a model
- Parse Summary

### Node Details

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  LangChain/OpenAI node used to send email content to an OpenAI model for summarization, classification, extraction, or routing.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the exact model, prompt, temperature, response format, and system/user instructions are not visible. Based on the workflow title, this node is intended to use GPT-4.
- **Key expressions or variables used:**  
  Not shown, but likely includes content from **Extract Email Fields**, such as subject and email body.  
  A robust implementation would ask the model to return strict JSON, for example:
  - summary
  - detected department
  - priority
  - action recommendation
- **Input and output connections:**  
  - Input: **Extract Email Fields**
  - Output: **Parse Summary**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`. Requires a compatible LangChain/OpenAI node version and valid OpenAI credentials in n8n.
- **Edge cases or potential failure types:**  
  - OpenAI authentication or quota failure
  - Model name unavailable in account/region
  - Timeout or rate limit
  - Non-deterministic output format
  - Response not matching expected JSON schema
  - Prompt injection from email contents if not handled carefully
- **Sub-workflow reference:**  
  None

#### Parse Summary
- **Type and technical role:** `n8n-nodes-base.code`  
  A Code node that parses the model response and transforms it into stable structured fields.
- **Configuration choices:**  
  The actual JavaScript code is not present in the JSON excerpt. This node likely:
  - reads the model output text
  - parses JSON or splits structured text
  - maps values into fields such as department, summary, priority, and status
- **Key expressions or variables used:**  
  Not shown. Typical code references the previous node output, for example model response content.
- **Input and output connections:**  
  - Input: **Message a model**
  - Output: **Prepare Excel Row**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`. Ensure the code syntax matches the current n8n Code node runtime.
- **Edge cases or potential failure types:**  
  - `JSON.parse` failure if the AI response is not valid JSON
  - Missing expected keys
  - Null/undefined values
  - Multi-item handling issues if the trigger emits batches
  - Inconsistent model format across executions
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Logging Preparation and Persistence

### Overview
This block converts the parsed AI result into a structured row for Excel and writes it into a Microsoft Excel 365 table for persistent logging.

### Nodes Involved
- Prepare Excel Row
- Append rows to table

### Node Details

#### Prepare Excel Row
- **Type and technical role:** `n8n-nodes-base.set`  
  A Set node used to shape the parsed data into the exact column structure expected by the Excel table.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the specific columns are not visible. Likely fields include:
  - timestamp
  - sender
  - subject
  - summary
  - department
  - priority
  - original message ID
  - validation status
- **Key expressions or variables used:**  
  Not shown. Likely pulls values from **Parse Summary** and possibly original email metadata from earlier nodes.
- **Input and output connections:**  
  - Input: **Parse Summary**
  - Outputs:
    - **Append rows to table**
    - **Lookup Department**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - Column names not aligned with the destination Excel table
  - Empty or malformed values
  - Date values not in Excel-compatible format
  - Accidental loss of fields if “keep only set fields” is enabled
- **Sub-workflow reference:**  
  None

#### Append rows to table
- **Type and technical role:** `n8n-nodes-base.microsoftExcel`  
  Microsoft Excel 365 node used to append a new record to an Excel table.
- **Configuration choices:**  
  Parameters are empty in the JSON, so workbook, file location, worksheet, table name, and column mappings are not visible. The node name indicates it performs an append operation.
- **Key expressions or variables used:**  
  Not shown. Typically each Excel column is mapped from fields produced by **Prepare Excel Row**.
- **Input and output connections:**  
  - Input: **Prepare Excel Row**
  - Output: none shown
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`. Requires Microsoft credentials with access to the target workbook in OneDrive or SharePoint.
- **Edge cases or potential failure types:**  
  - Microsoft OAuth authentication failure
  - Workbook/table not found
  - Table locked or unavailable
  - Column mismatch
  - Invalid data types
  - Excel API throttling
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: Department Validation and Routing

### Overview
This block checks whether the department inferred from the email exists in a reference data source, then conditionally sends alerts and follow-up communication.

### Nodes Involved
- Lookup Department
- Validate Department
- Send Notification
- Send email

### Node Details

#### Lookup Department
- **Type and technical role:** `n8n-nodes-base.microsoftExcel`  
  Microsoft Excel 365 node used as a lookup/search node against a workbook table containing department references.
- **Configuration choices:**  
  Parameters are empty in the JSON. It likely searches an Excel table for the department field produced by **Prepare Excel Row**.
- **Key expressions or variables used:**  
  Not shown. Likely uses the detected department from AI parsing as the search key.
- **Input and output connections:**  
  - Input: **Prepare Excel Row**
  - Output: **Validate Department**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - Lookup table not found
  - No match returned
  - Multiple matches returned
  - Search field case sensitivity or formatting mismatch
  - Authentication/API quota issues
- **Sub-workflow reference:**  
  None

#### Validate Department
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional logic node that checks whether the lookup result satisfies expected validation criteria.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the exact IF condition is unknown. Since only one outgoing branch is present in the exported connections, the visible configuration suggests only one branch is wired downstream. In practice this likely means:
  - “if department exists” then notify/send
  - unmatched branch currently unused
- **Key expressions or variables used:**  
  Not shown. Likely checks presence of a row, department value equality, or a non-empty result set.
- **Input and output connections:**  
  - Input: **Lookup Department**
  - Output: visible connected branch to:
    - **Send Notification**
    - **Send email**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - Condition based on a field that may not exist
  - Unexpected array/object shape from Excel lookup
  - False positives if lookup returns approximate or partial matches
- **Sub-workflow reference:**  
  None

#### Send Notification
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Telegram node used to send a notification message, likely to an internal channel or user.
- **Configuration choices:**  
  Parameters are empty in the JSON. The node likely sends a message containing subject, sender, summary, and department.
- **Key expressions or variables used:**  
  Not shown. Typical fields include a formatted summary built from validated data.
- **Input and output connections:**  
  - Input: **Validate Department**
  - Output: none shown
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`. Requires Telegram bot credentials and a target chat ID.
- **Edge cases or potential failure types:**  
  - Invalid bot token
  - Bot not allowed in target chat
  - Wrong chat ID
  - Telegram rate limiting
  - Message formatting errors if Markdown/HTML mode is used
- **Sub-workflow reference:**  
  None

#### Send email
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends an outbound email, likely for forwarding, escalation, or acknowledgment.
- **Configuration choices:**  
  Parameters are empty in the JSON, so SMTP host, sender, recipient, subject, and body template are not visible.
- **Key expressions or variables used:**  
  Not shown. Likely uses fields from the validated email summary.
- **Input and output connections:**  
  - Input: **Validate Department**
  - Output: none shown
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`. Requires SMTP or service-specific email credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - SMTP authentication error
  - Blocked sender domain
  - Recipient missing or invalid
  - TLS/port mismatch
  - Email content template referencing missing fields
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: Annotation / Canvas Notes

### Overview
These nodes are visual documentation elements placed on the canvas. In this workflow export, all sticky notes are empty, so they provide no usable operational metadata.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation node only; no execution role.
- **Configuration choices:**  
  Content is empty.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** none for runtime
- **Sub-workflow reference:** None

#### Sticky Note1
- Same role as above; content is empty.

#### Sticky Note2
- Same role as above; content is empty.

#### Sticky Note4
- Same role as above; content is empty.

#### Sticky Note5
- Same role as above; content is empty.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Email Trigger (IMAP) | IMAP Email Trigger | Starts workflow on incoming email |  | Extract Email Fields |  |
| Extract Email Fields | Set | Normalizes email fields for AI processing | Email Trigger (IMAP) | Message a model |  |
| Message a model | OpenAI / LangChain | Sends email content to GPT-style model for classification or summary | Extract Email Fields | Parse Summary |  |
| Parse Summary | Code | Parses model output into structured fields | Message a model | Prepare Excel Row |  |
| Prepare Excel Row | Set | Maps parsed data into Excel row schema | Parse Summary | Append rows to table; Lookup Department |  |
| Append rows to table | Microsoft Excel 365 | Appends structured log row into Excel table | Prepare Excel Row |  |  |
| Lookup Department | Microsoft Excel 365 | Searches department reference data | Prepare Excel Row | Validate Department |  |
| Validate Department | If | Checks whether lookup result satisfies validation rule | Lookup Department | Send Notification; Send email |  |
| Send Notification | Telegram | Sends Telegram alert | Validate Department |  |  |
| Send email | Email Send | Sends follow-up or routing email | Validate Department |  |  |
| Sticky Note | Sticky Note | Canvas annotation only |  |  |  |
| Sticky Note1 | Sticky Note | Canvas annotation only |  |  |  |
| Sticky Note2 | Sticky Note | Canvas annotation only |  |  |  |
| Sticky Note4 | Sticky Note | Canvas annotation only |  |  |  |
| Sticky Note5 | Sticky Note | Canvas annotation only |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

Because the JSON does not include the detailed node parameters, the steps below reconstruct the workflow architecture and the likely intended setup. You will need to choose exact field names, prompts, workbook locations, and notification text during implementation.

1. **Create a new workflow**
   - Name it something like: **AI Email Inbox → Excel 365 Smart Logger**
   - Keep execution order as default unless you specifically need `v1` compatibility.

2. **Add an IMAP trigger node**
   - Node type: **Email Trigger (IMAP)** / `Email Read IMAP`
   - Configure IMAP credentials:
     - host
     - port
     - username
     - password or app password
     - TLS/SSL settings
   - Choose the mailbox/folder to monitor, such as `INBOX`
   - Configure message polling or unread behavior depending on your email server
   - Decide whether to mark messages as read after processing
   - Test with a sample incoming email

3. **Add a Set node named `Extract Email Fields`**
   - Connect it after the IMAP trigger
   - Add fields you want downstream, for example:
     - `from`
     - `fromName`
     - `subject`
     - `text`
     - `html`
     - `receivedAt`
     - `messageId`
   - Use expressions pulling data from the IMAP node output
   - If possible, preserve the original body text in a plain-text field for LLM use
   - Decide whether to keep only the fields you explicitly set

4. **Add an OpenAI/LangChain node named `Message a model`**
   - Connect it after `Extract Email Fields`
   - Configure OpenAI credentials
   - Select a model appropriate for structured extraction, such as a GPT-4-class model available in your account
   - Build a prompt that asks the model to analyze the email and return strict JSON
   - Recommended requested fields:
     - `summary`
     - `department`
     - `priority`
     - `sentiment`
     - `action`
   - Example prompt design:
     - include subject
     - include sender
     - include body text
     - instruct the model to return JSON only
   - Keep temperature low for predictable formatting

5. **Add a Code node named `Parse Summary`**
   - Connect it after `Message a model`
   - Write code to:
     - read the model response text
     - parse JSON safely
     - provide fallback values if parsing fails
   - Example logic:
     - attempt `JSON.parse(response)`
     - if parsing fails, set:
       - `summary` to raw response
       - `department` to `Unknown`
       - `priority` to `Normal`
   - Return one normalized item per email

6. **Add a Set node named `Prepare Excel Row`**
   - Connect it after `Parse Summary`
   - Define the output schema to match your Excel logging table
   - Suggested fields:
     - `timestamp`
     - `sender`
     - `subject`
     - `summary`
     - `department`
     - `priority`
     - `messageId`
     - `status`
   - Set `timestamp` from current execution time or original email date
   - Ensure names exactly match your Excel table columns if you want simple mappings

7. **Add a Microsoft Excel 365 node named `Append rows to table`**
   - Connect it from `Prepare Excel Row`
   - Configure Microsoft credentials:
     - Microsoft 365 OAuth2
     - access to OneDrive or SharePoint location
   - Choose operation equivalent to **append row(s)**
   - Select:
     - workbook
     - worksheet
     - table
   - Map each field from `Prepare Excel Row` to the Excel columns
   - Test by writing a sample row

8. **Add a second Microsoft Excel 365 node named `Lookup Department`**
   - Also connect it from `Prepare Excel Row`
   - Configure it to search a reference table containing valid departments
   - Choose the workbook and table where department metadata is stored
   - Search using the `department` field created by AI parsing
   - Return matching row(s)

9. **Add an IF node named `Validate Department`**
   - Connect it after `Lookup Department`
   - Configure a condition such as:
     - department lookup returned at least one row
     - or a returned `DepartmentName` equals the parsed `department`
   - In the provided workflow export, only one branch is connected downstream
   - Recommended setup:
     - **true** branch = valid department found
     - **false** branch = optionally log error, fallback route, or send a warning

10. **Add a Telegram node named `Send Notification`**
    - Connect it from the validation success branch
    - Configure Telegram bot credentials
    - Set the target chat ID
    - Use a formatted message, for example:
      - subject
      - sender
      - detected department
      - summary
      - priority
    - Test that the bot can post to the desired chat/channel

11. **Add an Email Send node named `Send email`**
    - Connect it from the same validation success branch
    - Configure SMTP or another supported email credential
    - Set:
      - from address
      - to address or dynamic recipient
      - subject
      - body
    - Suggested body content:
      - original sender
      - original subject
      - AI summary
      - department
      - any routing instructions

12. **Verify the connection layout**
    - `Email Trigger (IMAP)` → `Extract Email Fields`
    - `Extract Email Fields` → `Message a model`
    - `Message a model` → `Parse Summary`
    - `Parse Summary` → `Prepare Excel Row`
    - `Prepare Excel Row` → `Append rows to table`
    - `Prepare Excel Row` → `Lookup Department`
    - `Lookup Department` → `Validate Department`
    - `Validate Department` → `Send Notification`
    - `Validate Department` → `Send email`

13. **Add optional error handling**
    - Since the current workflow has no visible error branch, consider adding:
      - parse failure logging
      - fallback department route
      - notification when Excel append fails
      - dead-letter logging table
      - deduplication by message ID

14. **Test with realistic emails**
    - Use:
      - one email that should map to a valid department
      - one email with vague language
      - one malformed or empty-body message
      - one email producing no department match
    - Confirm:
      - AI output is parseable
      - Excel row is appended correctly
      - lookup behaves as intended
      - Telegram and email actions trigger only when expected

15. **Activate the workflow**
    - Once all credentials and mappings are verified, activate the workflow
    - Monitor first runs closely for:
      - duplicate trigger behavior
      - invalid AI JSON
      - Excel throttling
      - notification formatting problems

### Credential Configuration Summary

- **IMAP credentials**
  - Email server host/port
  - Username/password or app password
  - SSL/TLS enabled if required

- **OpenAI credentials**
  - API key
  - Access to the selected model

- **Microsoft Excel 365 credentials**
  - OAuth2 authorization
  - Workbook access permissions in OneDrive/SharePoint

- **Telegram credentials**
  - Bot token
  - Chat ID

- **Email Send credentials**
  - SMTP server or provider-specific setup
  - Sender mailbox access

### Sub-workflow Setup
This workflow does **not** include any Execute Workflow or sub-workflow invocation nodes. No sub-workflow setup is required.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title in the provided export: “AI Email Inbox → Excel 365 Smart Logger” | Internal workflow naming |
| User-provided title: “Route and log incoming emails with GPT-4, Excel 365 and Telegram” | Descriptive label for the same process |
| All sticky note nodes in the provided JSON are empty | No additional visual documentation was embedded in the canvas |
| Many node parameter objects are empty in the provided JSON excerpt | Exact prompts, mappings, workbook names, and notification templates must be rebuilt manually |

## Additional Implementation Notes

- The workflow is currently **inactive** in the provided export.
- There is **one explicit trigger entry point**: `Email Trigger (IMAP)`.
- The IF node appears to have only one connected branch in the exported structure. This is valid, but it means unmatched/false outcomes are not handled unless you add another branch.
- The design assumes the AI result is stable enough to drive Excel logging and department routing. In production, you should strongly prefer strict JSON output plus defensive parsing.
- If department routing is business-critical, consider replacing free-text department generation with a controlled classification prompt limited to a known list of departments.