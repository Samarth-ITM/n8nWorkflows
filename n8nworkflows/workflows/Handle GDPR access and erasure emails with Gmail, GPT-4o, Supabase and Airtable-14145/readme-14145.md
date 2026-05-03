Handle GDPR access and erasure emails with Gmail, GPT-4o, Supabase and Airtable

https://n8nworkflows.xyz/workflows/handle-gdpr-access-and-erasure-emails-with-gmail--gpt-4o--supabase-and-airtable-14145


# Handle GDPR access and erasure emails with Gmail, GPT-4o, Supabase and Airtable

# 1. Workflow Overview

This workflow automates the handling of GDPR Data Subject Requests received by email. It monitors a Gmail inbox, uses GPT-4o to classify each request as either an **access request** or an **erasure request**, retrieves matching personal data from **Supabase** and **Airtable**, generates a response email, optionally deletes the data, and logs the completed action to **Google Sheets** for audit purposes.

Typical use cases:
- Responding to GDPR Article 15 access requests
- Executing GDPR Article 17 erasure requests
- Maintaining an auditable record of request handling
- Coordinating data retrieval/deletion across multiple systems

## 1.1 Input Reception and Request Classification
The workflow starts when a new Gmail message arrives. An AI Agent analyzes the sender, subject, and body to determine the request type and extract the relevant email addresses.

## 1.2 Database Lookup and Consolidation
The workflow queries Supabase for rows matching the data subject’s email, then consolidates the results into a single item to avoid downstream duplication.

## 1.3 CRM Lookup and Consolidation
It then queries Airtable for the same email address and consolidates the CRM results into the unified request context.

## 1.4 Report Compilation
A second AI Agent transforms the retrieved data into a structured GDPR response payload, including an email subject and HTML body.

## 1.5 Decision and Response Execution
An IF node branches the flow:
- **Access**: send the compiled report by email
- **Erasure**: delete records from Supabase and Airtable, then send an erasure confirmation

## 1.6 Audit Logging
Both branches merge back together and append the request outcome to a Google Sheets audit log.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Request Classification

**Overview:**  
This block receives incoming DSR emails from Gmail and uses an AI agent backed by GPT-4o to classify the request. It extracts the request type, requester identity, and data subject email into a structured format for later processing.

**Nodes Involved:**  
- Gmail — DSR Inbox
- Classify DSR Request
- OpenAI — Classify
- DSR Classification Schema
- Overview
- Prerequisites
- Setup Required
- How It Works
- Classify note

### Node Details

#### Gmail — DSR Inbox
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`; trigger node that polls a Gmail mailbox for new incoming emails.
- **Configuration choices:**  
  - Polling frequency: every minute  
  - Spam/trash excluded
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point of the workflow  
  - Outputs to **Classify DSR Request**
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 not connected or expired  
  - Gmail polling quota/rate issues  
  - Emails without usable body text  
  - Delayed polling due to n8n execution scheduling
- **Sub-workflow reference:** None

#### Classify DSR Request
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI Agent that classifies the DSR and extracts fields from the email.
- **Configuration choices:**  
  - Prompt includes:
    - `from`
    - `subject`
    - `text` or fallback `snippet`
  - System prompt instructs the model to identify:
    - `access`
    - `erasure`
    - `unknown`
  - Structured parser enabled
  - Intermediate steps disabled
- **Key expressions or variables used:**  
  - `{{ $json.from }}`
  - `{{ $json.subject }}`
  - `{{ $json.text || $json.snippet }}`
- **Input and output connections:**  
  - Main input from **Gmail — DSR Inbox**
  - AI language model input from **OpenAI — Classify**
  - AI output parser input from **DSR Classification Schema**
  - Main output to **Query Supabase**
- **Version-specific requirements:** Type version `1.8`; requires compatible LangChain/AI nodes in the n8n instance.
- **Edge cases or potential failure types:**  
  - LLM misclassification on vague or multilingual emails  
  - Missing or malformed email fields  
  - Structured parser failure if model output does not match schema  
  - Ambiguous representation requests where requester and data subject differ
- **Sub-workflow reference:** None

#### OpenAI — Classify
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model provider for the classification agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects as AI language model to **Classify DSR Request**
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - OpenAI authentication failure  
  - Model availability issues  
  - Token/context limits for unusually long emails  
  - API latency or rate limiting
- **Sub-workflow reference:** None

#### DSR Classification Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; schema validator/parser for classification output.
- **Configuration choices:**  
  - Manual JSON schema requiring:
    - `request_type`
    - `requester_email`
    - `requester_name`
    - `data_subject_email`
    - `request_summary`
  - `request_type` enum restricted to `access`, `erasure`, `unknown`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects as AI output parser to **Classify DSR Request**
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - Missing required fields in model output  
  - Invalid enum value  
  - Empty `requester_name` or invalid email string still technically passing as plain string
- **Sub-workflow reference:** None

#### Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only node.
- **Configuration choices:** Describes the full purpose and flow.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Prerequisites
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only node.
- **Configuration choices:** Lists required services and credentials.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Setup Required
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only node.
- **Configuration choices:** Lists nodes requiring manual credential or resource setup.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### How It Works
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation-only node.
- **Configuration choices:** Explains the end-to-end logic in numbered steps.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Classify note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment for the classification block.
- **Configuration choices:** Notes the block’s purpose.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.2 Database Lookup and Consolidation

**Overview:**  
This block queries Supabase for all records associated with the extracted data subject email. The results are then collapsed into a single item containing shared request context plus the full set of database matches.

**Nodes Involved:**  
- Query Supabase
- Consolidate DB Results
- Supabase note
- Consolidate DB note

### Node Details

#### Query Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`; retrieves matching rows from the primary database.
- **Configuration choices:**  
  - Operation: `getAll`
  - Table: `users`
  - Return all matching rows
  - Filter condition:
    - `email eq {{$json.output.data_subject_email}}`
  - Retry enabled: 3 tries, 2 seconds apart
  - Error handling: continue to error output
- **Key expressions or variables used:**  
  - `{{ $json.output.data_subject_email }}`
- **Input and output connections:**  
  - Input from **Classify DSR Request**
  - Output to **Consolidate DB Results**
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Supabase credential failure  
  - Wrong table name or nonexistent `users` table  
  - Field mismatch if the email column is not named `email`  
  - If classification returns blank email, query may fail or return no results  
  - Because `continueErrorOutput` is enabled, downstream code must tolerate partial failures
- **Sub-workflow reference:** None

#### Consolidate DB Results
- **Type and technical role:** `n8n-nodes-base.code`; merges all Supabase items into one normalized payload.
- **Configuration choices:**  
  - Reads all input rows with `$input.all()`
  - Pulls classification context from **Classify DSR Request**
  - Excludes rows containing `json.error`
  - Creates:
    - `request_type`
    - `requester_email`
    - `requester_name`
    - `data_subject_email`
    - `request_summary`
    - `db_records`
    - `db_record_count`
- **Key expressions or variables used:**  
  - `$('Classify DSR Request').first().json.output`
  - `$input.all()`
- **Input and output connections:**  
  - Input from **Query Supabase**
  - Output to **Query Airtable CRM**
- **Version-specific requirements:** Type version `2` for the Code node syntax used.
- **Edge cases or potential failure types:**  
  - If **Classify DSR Request** output is absent, fields may become undefined  
  - If Supabase error items are the only outputs, `db_records` becomes an empty array  
  - Any typo in node names would break the `$()` lookup
- **Sub-workflow reference:** None

#### Supabase note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains that the users table is queried by email.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Consolidate DB note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains why results are merged into a single item.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.3 CRM Lookup and Consolidation

**Overview:**  
This block searches Airtable for CRM entries matching the data subject email. It merges the CRM results into the existing request context so the reporting step can use a single combined payload.

**Nodes Involved:**  
- Query Airtable CRM
- Consolidate CRM Results
- Airtable note
- Consolidate CRM note

### Node Details

#### Query Airtable CRM
- **Type and technical role:** `n8n-nodes-base.airtable`; searches CRM records by formula.
- **Configuration choices:**  
  - Operation: `search`
  - Base ID: placeholder `YOUR_BASE_ID`
  - Table name: placeholder `YOUR_TABLE_NAME`
  - `filterByFormula` dynamically built as:
    - `{Email}='data_subject_email'`
  - Retry enabled: 3 tries, 2 seconds apart
  - Error handling: continue on error
- **Key expressions or variables used:**  
  - `{{ `{Email}='${$json.data_subject_email}'` }}`
- **Input and output connections:**  
  - Input from **Consolidate DB Results**
  - Output to **Consolidate CRM Results**
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**  
  - Invalid Airtable credential or PAT scope issues  
  - Wrong base/table configuration  
  - Formula failure if the field is not actually named `Email`  
  - Apostrophes or special characters in email values can break formula syntax  
  - Empty email returns zero results or formula error
- **Sub-workflow reference:** None

#### Consolidate CRM Results
- **Type and technical role:** `n8n-nodes-base.code`; combines Airtable matches with the existing request and database context.
- **Configuration choices:**  
  - Reads all Airtable output items
  - Retrieves the previous context from **Consolidate DB Results**
  - Ignores items with `json.error`
  - Prefers `r.json.fields` when present, otherwise `r.json`
  - Produces:
    - all previous fields
    - `crm_records`
    - `crm_record_count`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Consolidate DB Results').first().json`
- **Input and output connections:**  
  - Input from **Query Airtable CRM**
  - Output to **Compile Data Report**
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - Missing upstream context causes failure in named node lookup  
  - Airtable schema differences may lead to inconsistent `fields` payloads  
  - If all results are errors, CRM arrays become empty
- **Sub-workflow reference:** None

#### Airtable note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains use of `filterByFormula`.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Consolidate CRM note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains the deduplication/merge purpose.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.4 Report Compilation

**Overview:**  
This block asks GPT-4o to compile the collected records into a GDPR-ready response. It returns structured fields containing an email subject, HTML email body, and metadata about the data found.

**Nodes Involved:**  
- Compile Data Report
- OpenAI — Compile
- Report Schema
- Compile note

### Node Details

#### Compile Data Report
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI Agent that transforms raw records into a user-facing compliance response.
- **Configuration choices:**  
  - Prompt includes:
    - request type
    - requester
    - data subject email
    - request summary
    - stringified database records
    - stringified CRM records
  - System message instructs the model to:
    - produce Article 15 access-style reporting
    - produce Article 17 erasure confirmation
    - output valid HTML
  - Structured output parser enabled
- **Key expressions or variables used:**  
  - `{{ $json.request_type }}`
  - `{{ $json.data_subject_email }}`
  - `{{ $json.requester_name }}`
  - `{{ $json.requester_email }}`
  - `{{ $json.request_summary }}`
  - `{{ $json.db_record_count }}`
  - `{{ JSON.stringify($json.db_records, null, 2) }}`
  - `{{ $json.crm_record_count }}`
  - `{{ JSON.stringify($json.crm_records, null, 2) }}`
- **Input and output connections:**  
  - Main input from **Consolidate CRM Results**
  - AI language model input from **OpenAI — Compile**
  - AI output parser from **Report Schema**
  - Main output to **Access or Erasure?**
- **Version-specific requirements:** Type version `1.8`
- **Edge cases or potential failure types:**  
  - Excessively large record payloads may exceed model token limits  
  - Invalid HTML or overly verbose model response  
  - Model may overstate certainty if data retrieval failed silently upstream  
  - Erasure branch email is generated before deletion outcome is verified
- **Sub-workflow reference:** None

#### OpenAI — Compile
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backing the report generation agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects as AI language model to **Compile Data Report**
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - OpenAI auth, rate limit, or availability issues  
  - Token overflow if input data becomes too large
- **Sub-workflow reference:** None

#### Report Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; structured parser for the report payload.
- **Configuration choices:**  
  - Requires:
    - `email_subject`
    - `email_body_html`
    - `data_found`
    - `systems_checked`
  - Optional:
    - `erasure_scope`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connects as AI output parser to **Compile Data Report**
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**  
  - Missing required keys from LLM output  
  - `systems_checked` not returned as array  
  - HTML body returned as malformed or incomplete string
- **Sub-workflow reference:** None

#### Compile note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains that the AI generates a legally compliant report.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.5 Decision and Response Execution

**Overview:**  
This block routes execution based on the classified request type. Access requests immediately send the generated report, while erasure requests perform deletion in Supabase and Airtable before emailing confirmation.

**Nodes Involved:**  
- Access or Erasure?
- Send Access Report
- Delete from Supabase
- Delete from Airtable
- Send Erasure Confirmation
- Route note
- Delete DB note

### Node Details

#### Access or Erasure?
- **Type and technical role:** `n8n-nodes-base.if`; conditional branch selector.
- **Configuration choices:**  
  - Loose type validation
  - Condition:
    - `$('Consolidate CRM Results').first().json.request_type == 'access'`
  - True branch = access path
  - False branch = erasure/other path
- **Key expressions or variables used:**  
  - `{{ $('Consolidate CRM Results').first().json.request_type }}`
- **Input and output connections:**  
  - Input from **Compile Data Report**
  - True output to **Send Access Report**
  - False output to **Delete from Supabase**
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**  
  - `unknown` request types are implicitly treated like erasure because only `access` is explicitly checked  
  - Missing `request_type` sends execution to false branch  
  - This is a significant logic risk for ambiguous emails
- **Sub-workflow reference:** None

#### Send Access Report
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the generated access report email.
- **Configuration choices:**  
  - To: requester email from consolidated context
  - Subject/body from **Compile Data Report** output
  - Retry enabled: 3 tries, 2 seconds apart
  - Continue on error enabled
- **Key expressions or variables used:**  
  - `{{ $('Consolidate CRM Results').first().json.requester_email }}`
  - `{{ $json.output.email_body_html }}`
  - `{{ $json.output.email_subject }}`
- **Input and output connections:**  
  - Input from true branch of **Access or Erasure?**
  - Output to **Merge Paths**
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**  
  - Gmail send permission issues  
  - HTML body rendering quirks  
  - Requester email may be missing or extracted incorrectly  
  - Because continue-on-error is enabled, audit logging may still mark the request completed even if email sending failed
- **Sub-workflow reference:** None

#### Delete from Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`; deletes matching user rows from Supabase.
- **Configuration choices:**  
  - Operation: `delete`
  - Table: `users`
  - Filter: `email eq data_subject_email`
  - Retry enabled: 3 tries
  - Continue on error enabled
- **Key expressions or variables used:**  
  - `{{ $('Consolidate CRM Results').first().json.data_subject_email }}`
- **Input and output connections:**  
  - Input from false branch of **Access or Erasure?**
  - Output to **Delete from Airtable**
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - Dangerous if an `unknown` request reaches this branch  
  - Wrong filter column or broad email matching can remove unintended rows  
  - Supabase permissions may allow read but not delete
- **Sub-workflow reference:** None

#### Delete from Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`; deletes Airtable records.
- **Configuration choices:**  
  - Operation: `delete`
  - Base/table placeholders remain to be replaced
  - Retry enabled: 3 tries
  - Continue on error enabled
- **Key expressions or variables used:** None visible in parameters
- **Input and output connections:**  
  - Input from **Delete from Supabase**
  - Output to **Send Erasure Confirmation**
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**  
  - As configured, there is no visible record ID mapping or search-to-delete linkage in this node definition  
  - Airtable delete operations typically require record IDs; without them, this node may fail or do nothing depending on runtime expectations  
  - Continue-on-error means the workflow may still send a deletion confirmation even if Airtable deletion failed
- **Sub-workflow reference:** None

#### Send Erasure Confirmation
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the erasure confirmation email.
- **Configuration choices:**  
  - To: requester email
  - Subject/body taken from **Compile Data Report**
  - Retry enabled: 3 tries
  - Continue on error enabled
- **Key expressions or variables used:**  
  - `{{ $('Consolidate CRM Results').first().json.requester_email }}`
  - `{{ $('Compile Data Report').first().json.output.email_body_html }}`
  - `{{ $('Compile Data Report').first().json.output.email_subject }}`
- **Input and output connections:**  
  - Input from **Delete from Airtable**
  - Output to **Merge Paths**
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**  
  - Confirmation may be sent even if one or both deletions failed  
  - Missing requester email  
  - Email send issues similar to access branch
- **Sub-workflow reference:** None

#### Route note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Describes branch routing purpose.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Delete DB note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** States that Supabase records are deleted in the erasure path.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.6 Audit Logging

**Overview:**  
This block recombines the access and erasure paths and writes a completion record to Google Sheets. It acts as the final compliance log for later audit or reporting.

**Nodes Involved:**  
- Merge Paths
- Log Audit Trail
- Merge note
- Audit note

### Node Details

#### Merge Paths
- **Type and technical role:** `n8n-nodes-base.merge`; merges the two route outputs into a single downstream flow.
- **Configuration choices:**  
  - Mode: `chooseBranch`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input 1 from **Send Access Report**
  - Input 2 from **Send Erasure Confirmation**
  - Output to **Log Audit Trail**
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**  
  - Behavior depends on which branch executes, which is appropriate here  
  - If an upstream branch errors but continues with malformed data, logging still proceeds
- **Sub-workflow reference:** None

#### Log Audit Trail
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends or updates an audit row in Google Sheets.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Document ID placeholder: `YOUR_AUDIT_SHEET_ID`
  - Sheet name: `DSR Audit Log`
  - Explicit column mapping:
    - Status = `completed`
    - Summary
    - Timestamp
    - Data Subject
    - Request Type
    - Requester Email
    - DB Records Found
    - CRM Records Found
  - Retry enabled: 3 tries
  - Continue on error enabled
- **Key expressions or variables used:**  
  - `{{ $('Consolidate CRM Results').first().json.request_summary }}`
  - `{{ DateTime.now().toISO() }}`
  - `{{ $('Consolidate CRM Results').first().json.data_subject_email }}`
  - `{{ $('Consolidate CRM Results').first().json.request_type }}`
  - `{{ $('Consolidate CRM Results').first().json.requester_email }}`
  - `{{ $('Consolidate CRM Results').first().json.db_record_count }}`
  - `{{ $('Consolidate CRM Results').first().json.crm_record_count }}`
- **Input and output connections:**  
  - Input from **Merge Paths**
  - No downstream node
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**  
  - Wrong spreadsheet ID or missing worksheet tab  
  - OAuth scope issues  
  - `appendOrUpdate` may behave unexpectedly if no matching key columns are configured externally  
  - Marks everything as `completed` regardless of partial upstream failures
- **Sub-workflow reference:** None

#### Merge note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Notes that both branches rejoin for logging.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Audit note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; contextual comment.
- **Configuration choices:** Explains the purpose of the Google Sheets log.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documents overall workflow purpose and flow |  |  | ## GDPR Data Subject Request Handler<br>Version 1.0.0 — Compliance<br><br>Processes inbound GDPR access/erasure requests from Gmail. Classifies the request with AI, queries Supabase (primary DB) and Airtable (CRM), compiles a full data report, sends the response to the data subject, and logs the completion to Google Sheets for audit.<br><br>Flow: Gmail Trigger => Classify Request (AI Agent) => Query Supabase => Consolidate DB => Query Airtable => Consolidate CRM => Compile Report (AI Agent) => Access or Erasure? (IF) => Send Access Report / Execute Erasure => Log Audit Trail |
| Prerequisites | Sticky Note | Lists required accounts, systems, and credentials |  |  | ## Prerequisites<br>- Gmail account for incoming DSR emails (Gmail OAuth2 credential)<br>- OpenAI API key (GPT-4o)<br>- Supabase project with users table (Supabase API credential)<br>- Airtable base with a Contacts table (Airtable Personal Access Token)<br>- Google Sheets for audit log (Google Sheets OAuth2)<br>- Gmail for sending responses (reuse inbox or separate send-only account) |
| Setup Required | Sticky Note | Lists manual setup actions |  |  | ## Setup Required<br>1. Gmail Trigger — connect Gmail OAuth2 credential<br>2. Classify DSR Request — connect OpenAI credential<br>3. Query Supabase — connect Supabase API credential, update table name 'users'<br>4. Query Airtable — connect Airtable credential, replace YOUR_BASE_ID and YOUR_TABLE_NAME<br>5. Compile Report — connect OpenAI credential<br>6. Send Access Report / Erasure Notice — connect Gmail OAuth2 credential<br>7. Log Audit Trail — connect Google Sheets credential, replace YOUR_AUDIT_SHEET_ID |
| How It Works | Sticky Note | Describes the workflow sequence |  |  | ## How It Works<br>1. Gmail Trigger fires when a new email arrives<br>2. AI Agent classifies: access or erasure, extracts requester and data subject email<br>3. Supabase queried for all rows matching data subject email; Code consolidates to 1 item<br>4. Airtable CRM queried for matching contact; Code consolidates to 1 item<br>5. Second AI Agent compiles all found data into a human-readable GDPR report<br>6. IF node routes: access => send report email; erasure => delete from both systems then send confirmation<br>7. Audit trail logged to Google Sheets with timestamp, request type, and outcome |
| Gmail — DSR Inbox | Gmail Trigger | Receives new DSR emails |  | Classify DSR Request |  |
| Classify note | Sticky Note | Documents classification block |  |  | AI classifies request type and extracts emails from raw email body |
| Classify DSR Request | LangChain Agent | Classifies request and extracts structured fields | Gmail — DSR Inbox; OpenAI — Classify; DSR Classification Schema | Query Supabase | AI classifies request type and extracts emails from raw email body |
| OpenAI — Classify | OpenAI Chat Model | LLM provider for classification |  | Classify DSR Request | AI classifies request type and extracts emails from raw email body |
| DSR Classification Schema | Structured Output Parser | Enforces classification output schema |  | Classify DSR Request | AI classifies request type and extracts emails from raw email body |
| Supabase note | Sticky Note | Documents Supabase lookup |  |  | Query users table for all rows matching data subject email |
| Query Supabase | Supabase | Fetches matching database rows | Classify DSR Request | Consolidate DB Results | Query users table for all rows matching data subject email |
| Consolidate DB note | Sticky Note | Documents Supabase result consolidation |  |  | Merges all Supabase rows into a single item to prevent duplication downstream |
| Consolidate DB Results | Code | Normalizes DB results into one item | Query Supabase | Query Airtable CRM | Merges all Supabase rows into a single item to prevent duplication downstream |
| Airtable note | Sticky Note | Documents Airtable lookup |  |  | Query CRM Contacts table for matching email via filterByFormula |
| Query Airtable CRM | Airtable | Searches CRM records by email | Consolidate DB Results | Consolidate CRM Results | Query CRM Contacts table for matching email via filterByFormula |
| Consolidate CRM note | Sticky Note | Documents CRM result consolidation |  |  | Merges all Airtable rows into a single item for the AI agent |
| Consolidate CRM Results | Code | Merges CRM data into unified context | Query Airtable CRM | Compile Data Report | Merges all Airtable rows into a single item for the AI agent |
| Compile note | Sticky Note | Documents report generation |  |  | AI compiles all data sources into a legally-compliant GDPR report email |
| Compile Data Report | LangChain Agent | Generates GDPR response subject/body | Consolidate CRM Results; OpenAI — Compile; Report Schema | Access or Erasure? | AI compiles all data sources into a legally-compliant GDPR report email |
| OpenAI — Compile | OpenAI Chat Model | LLM provider for report generation |  | Compile Data Report | AI compiles all data sources into a legally-compliant GDPR report email |
| Report Schema | Structured Output Parser | Enforces report output schema |  | Compile Data Report | AI compiles all data sources into a legally-compliant GDPR report email |
| Route note | Sticky Note | Documents branch routing |  |  | Routes to access response path or erasure execution path |
| Access or Erasure? | IF | Chooses access vs erasure path | Compile Data Report | Send Access Report; Delete from Supabase | Routes to access response path or erasure execution path |
| Send Access Report | Gmail | Sends access response email | Access or Erasure? | Merge Paths |  |
| Delete DB note | Sticky Note | Documents Supabase deletion |  |  | Deletes data subject records from Supabase (primary DB) |
| Delete from Supabase | Supabase | Deletes matching DB records | Access or Erasure? | Delete from Airtable | Deletes data subject records from Supabase (primary DB) |
| Delete from Airtable | Airtable | Deletes matching CRM records | Delete from Supabase | Send Erasure Confirmation |  |
| Send Erasure Confirmation | Gmail | Sends erasure confirmation email | Delete from Airtable | Merge Paths |  |
| Merge note | Sticky Note | Documents path merge |  |  | Merges both paths back to a single flow for audit logging |
| Merge Paths | Merge | Rejoins access and erasure branches | Send Access Report; Send Erasure Confirmation | Log Audit Trail | Merges both paths back to a single flow for audit logging |
| Audit note | Sticky Note | Documents audit logging |  |  | Logs every DSR to Google Sheets for GDPR-required audit trail |
| Log Audit Trail | Google Sheets | Writes final audit row | Merge Paths |  | Logs every DSR to Google Sheets for GDPR-required audit trail |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Handle GDPR access and erasure emails with Gmail, GPT-4o, Supabase and Airtable**

2. **Add a Gmail Trigger node** named **Gmail — DSR Inbox**.  
   - Type: **Gmail Trigger**
   - Configure Gmail OAuth2 credentials
   - Polling: **Every Minute**
   - Disable **Include Spam/Trash**
   - This is the workflow entry point

3. **Add an AI Agent node** named **Classify DSR Request**.  
   - Type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Set prompt type to define your own prompt
   - Use this logic in the user prompt:
     - Include sender, subject, and email body
     - Ask the model to determine whether the email is:
       - `access`
       - `erasure`
       - `unknown`
     - Ask it to extract:
       - requester email
       - requester name
       - data subject email
       - request summary
   - Use expressions:
     - `{{ $json.from }}`
     - `{{ $json.subject }}`
     - `{{ $json.text || $json.snippet }}`
   - Set a system message instructing the model to behave as a GDPR compliance assistant
   - Enable structured output parsing

4. **Add an OpenAI Chat Model node** named **OpenAI — Classify**.  
   - Type: **OpenAI Chat Model**
   - Credentials: OpenAI API key
   - Model: **gpt-4o**
   - Temperature: **0**
   - Connect it to **Classify DSR Request** using the AI language model connection

5. **Add a Structured Output Parser node** named **DSR Classification Schema**.  
   - Type: **Structured Output Parser**
   - Use a manual JSON schema with these required properties:
     - `request_type` as string enum: `access`, `erasure`, `unknown`
     - `requester_email` as string
     - `requester_name` as string
     - `data_subject_email` as string
     - `request_summary` as string
   - Connect it to **Classify DSR Request** using the AI output parser connection

6. **Connect Gmail trigger to classification agent.**  
   - Main connection:
     - **Gmail — DSR Inbox** → **Classify DSR Request**

7. **Add a Supabase node** named **Query Supabase**.  
   - Type: **Supabase**
   - Credentials: Supabase API credential
   - Operation: **Get Many / Get All**
   - Table: `users`
   - Return all: enabled
   - Add filter:
     - Column: `email`
     - Condition: equals
     - Value: `{{ $json.output.data_subject_email }}`
   - Enable retry on failure:
     - Max tries: **3**
     - Wait between tries: **2000 ms**
   - Set error handling to continue on error

8. **Connect classification to Supabase lookup.**  
   - **Classify DSR Request** → **Query Supabase**

9. **Add a Code node** named **Consolidate DB Results**.  
   - Type: **Code**
   - Mode: JavaScript
   - Purpose: collapse multiple Supabase records into a single item
   - Use logic equivalent to:
     - collect all input rows
     - ignore rows with `error`
     - read classification result from **Classify DSR Request**
     - output one object containing:
       - request metadata
       - `db_records`
       - `db_record_count`
   - The code should reference:
     - `$input.all()`
     - `$('Classify DSR Request').first().json.output`

10. **Connect Supabase results to the consolidation node.**  
    - **Query Supabase** → **Consolidate DB Results**

11. **Add an Airtable node** named **Query Airtable CRM**.  
    - Type: **Airtable**
    - Credentials: Airtable Personal Access Token
    - Operation: **Search**
    - Base ID: replace with your actual base ID instead of `YOUR_BASE_ID`
    - Table name: replace `YOUR_TABLE_NAME`
    - Use `filterByFormula`:
      - `{{ `{Email}='${$json.data_subject_email}'` }}`
    - Enable retry on failure:
      - Max tries: **3**
      - Wait between tries: **2000 ms**
    - Continue on error enabled

12. **Connect DB consolidation to Airtable lookup.**  
    - **Consolidate DB Results** → **Query Airtable CRM**

13. **Add a Code node** named **Consolidate CRM Results**.  
    - Type: **Code**
    - Purpose: merge Airtable results into the existing request context
    - Logic:
      - collect all Airtable rows
      - ignore any items containing `error`
      - pull previous context from **Consolidate DB Results**
      - store CRM records in `crm_records`
      - count them in `crm_record_count`
      - if Airtable returns objects with a `fields` property, use `fields`
    - Reference:
      - `$input.all()`
      - `$('Consolidate DB Results').first().json`

14. **Connect Airtable lookup to CRM consolidation.**  
    - **Query Airtable CRM** → **Consolidate CRM Results**

15. **Add another AI Agent node** named **Compile Data Report**.  
    - Type: **AI Agent**
    - Purpose: generate a GDPR response payload
    - Prompt should include:
      - request type
      - data subject email
      - requester name and email
      - request summary
      - all DB records as formatted JSON
      - all CRM records as formatted JSON
    - Ask for:
      - email subject line
      - full HTML email body
      - whether data was found
      - systems checked
      - erasure scope if relevant
    - System message should instruct the model to act like a GDPR/DPO assistant and generate valid HTML

16. **Add an OpenAI Chat Model node** named **OpenAI — Compile**.  
    - Credentials: OpenAI API key
   - Model: **gpt-4o**
   - Temperature: **0.2**
   - Connect it to **Compile Data Report** as the AI language model

17. **Add a Structured Output Parser node** named **Report Schema**.  
   - Configure a manual JSON schema requiring:
     - `email_subject` as string
     - `email_body_html` as string
     - `data_found` as boolean
     - `systems_checked` as array of strings
   - Add optional:
     - `erasure_scope` as string
   - Connect it to **Compile Data Report** as the output parser

18. **Connect CRM consolidation to report compilation.**  
   - **Consolidate CRM Results** → **Compile Data Report**

19. **Add an IF node** named **Access or Erasure?**  
   - Condition:
     - left value: `{{ $('Consolidate CRM Results').first().json.request_type }}`
     - operator: equals
     - right value: `access`
   - Leave loose type validation enabled if matching the original setup
   - True branch = access
   - False branch = erasure/other

20. **Connect report compilation to the IF node.**  
   - **Compile Data Report** → **Access or Erasure?**

21. **Add a Gmail node** named **Send Access Report**.  
   - Type: **Gmail**
   - Operation: send email
   - Credentials: Gmail OAuth2
   - To: `{{ $('Consolidate CRM Results').first().json.requester_email }}`
   - Subject: `{{ $json.output.email_subject }}`
   - Message/body: `{{ $json.output.email_body_html }}`
   - Continue on error: enabled
   - Retry on failure: 3 attempts, 2000 ms wait

22. **Connect the true branch of the IF node to Send Access Report.**

23. **Add a Supabase node** named **Delete from Supabase**.  
   - Type: **Supabase**
   - Operation: **Delete**
   - Table: `users`
   - Filter:
     - column `email`
     - equals `{{ $('Consolidate CRM Results').first().json.data_subject_email }}`
   - Continue on error enabled
   - Retry on failure enabled with 3 attempts and 2000 ms wait

24. **Connect the false branch of the IF node to Delete from Supabase.**

25. **Add an Airtable node** named **Delete from Airtable**.  
   - Type: **Airtable**
   - Operation: **Delete**
   - Use the same Airtable credentials, base, and table as above
   - Important: Airtable deletion usually requires record IDs. To make this branch actually work reliably, you will typically need either:
     - record IDs from the earlier Airtable search, or
     - a separate search/select step that maps found records to IDs before deletion
   - The provided workflow includes a delete node without visible record-ID mapping, so you should verify and complete this part in your environment

26. **Connect Delete from Supabase to Delete from Airtable.**

27. **Add a Gmail node** named **Send Erasure Confirmation**.  
   - Type: **Gmail**
   - Operation: send email
   - To: `{{ $('Consolidate CRM Results').first().json.requester_email }}`
   - Subject: `{{ $('Compile Data Report').first().json.output.email_subject }}`
   - Message/body: `{{ $('Compile Data Report').first().json.output.email_body_html }}`
   - Continue on error enabled
   - Retry on failure enabled with 3 attempts and 2000 ms wait

28. **Connect Delete from Airtable to Send Erasure Confirmation.**

29. **Add a Merge node** named **Merge Paths**.  
   - Type: **Merge**
   - Mode: **Choose Branch**
   - Connect:
     - **Send Access Report** → input 1
     - **Send Erasure Confirmation** → input 2

30. **Add a Google Sheets node** named **Log Audit Trail**.  
   - Type: **Google Sheets**
   - Credentials: Google Sheets OAuth2
   - Operation: **Append or Update**
   - Spreadsheet/document ID: replace `YOUR_AUDIT_SHEET_ID`
   - Sheet/tab name: `DSR Audit Log`
   - Map these columns manually:
     - `Status` = `completed`
     - `Summary` = `{{ $('Consolidate CRM Results').first().json.request_summary }}`
     - `Timestamp` = `{{ DateTime.now().toISO() }}`
     - `Data Subject` = `{{ $('Consolidate CRM Results').first().json.data_subject_email }}`
     - `Request Type` = `{{ $('Consolidate CRM Results').first().json.request_type }}`
     - `Requester Email` = `{{ $('Consolidate CRM Results').first().json.requester_email }}`
     - `DB Records Found` = `{{ $('Consolidate CRM Results').first().json.db_record_count }}`
     - `CRM Records Found` = `{{ $('Consolidate CRM Results').first().json.crm_record_count }}`
   - Continue on error enabled
   - Retry on failure enabled with 3 attempts and 2000 ms wait

31. **Connect Merge Paths to Log Audit Trail.**

32. **Add the sticky notes** if you want the same visual documentation inside n8n:
   - Overview
   - Prerequisites
   - Setup Required
   - How It Works
   - Classify note
   - Supabase note
   - Consolidate DB note
   - Airtable note
   - Consolidate CRM note
   - Compile note
   - Route note
   - Delete DB note
   - Merge note
   - Audit note

33. **Test with both request types.**
   - Send one email clearly asking for data access
   - Send one email clearly asking for deletion
   - Confirm:
     - classification output is valid
     - Supabase lookup works
     - Airtable formula matches your actual field names
     - access branch sends the report
     - erasure branch deletes the correct records
     - Google Sheets audit row is created

34. **Harden the logic before production.**  
   Recommended improvements based on the current design:
   - Add a dedicated branch for `unknown` instead of treating it as erasure
   - Validate requester authorization when requester and data subject differ
   - Confirm deletion success before sending erasure confirmation
   - Store deletion outcome in audit log instead of always writing `completed`
   - Escape Airtable formula values safely
   - Ensure record IDs are available for Airtable deletion
   - Consider adding human review for ambiguous or high-risk requests

### Required Credentials
- **Gmail OAuth2** for:
  - Gmail — DSR Inbox
  - Send Access Report
  - Send Erasure Confirmation
- **OpenAI API credential** for:
  - OpenAI — Classify
  - OpenAI — Compile
- **Supabase API credential** for:
  - Query Supabase
  - Delete from Supabase
- **Airtable credential / PAT** for:
  - Query Airtable CRM
  - Delete from Airtable
- **Google Sheets OAuth2** for:
  - Log Audit Trail

### Sub-workflow Setup
This workflow does **not** invoke any sub-workflows and does not expose itself as a callable sub-workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is documented internally with sticky notes covering purpose, prerequisites, setup, execution flow, routing, deletion, merge, and audit logging. | In-workflow documentation |
| Placeholders must be replaced before use: `YOUR_BASE_ID`, `YOUR_TABLE_NAME`, and `YOUR_AUDIT_SHEET_ID`. | Airtable and Google Sheets configuration |
| The current IF logic sends every non-`access` request, including `unknown`, into the deletion branch. This should be corrected before production use. | Critical implementation note |
| The Airtable deletion step is incomplete unless valid record IDs are supplied. | Critical implementation note |
| The workflow logs status as `completed` even when upstream nodes use continue-on-error and may have partially failed. | Audit/logging caveat |