Triage and escalate tenant complaints from Gmail or forms to Slack with Claude AI

https://n8nworkflows.xyz/workflows/triage-and-escalate-tenant-complaints-from-gmail-or-forms-to-slack-with-claude-ai-14292


# Triage and escalate tenant complaints from Gmail or forms to Slack with Claude AI

# 1. Workflow Overview

This workflow automates facility-management complaint handling for commercial tenants. It accepts complaints from either Gmail or a web form, uses Claude AI to classify and prioritize the issue, assigns a technician from Airtable, creates a work order, sends an acknowledgement email to the tenant, notifies the operations team in Slack, and separately monitors open tickets for SLA breaches.

It is designed for FM teams that need consistent triage, fast acknowledgement, structured ticket creation, and automatic escalation of overdue issues.

## 1.1 Complaint Ingestion

Two entry points collect complaints:

- a **Gmail Trigger** for unread inbox emails
- a **Webhook** for form submissions

Each input path is normalized into the same internal schema before being merged.

## 1.2 AI Triage and Ticket Enrichment

The merged complaint is sent to **Anthropic Claude Sonnet** for extraction and classification. The workflow then parses the model output, validates the JSON structure, generates a short ticket ID, and computes an SLA deadline based on the AI-assigned priority tier.

## 1.3 Technician Assignment and Work Order Creation

The classified fault category is used to search an Airtable technician directory. The matching technician is inserted into a new Airtable complaint/work-order record along with complaint details, SLA, tone, recurrence, and tenant information.

## 1.4 Tenant Acknowledgement and Team Dispatch

After the work order is created, the workflow sends the tenant an acknowledgement email via Gmail, updates the Airtable record to indicate the acknowledgement was sent, and posts a structured Slack message for the FM team.

## 1.5 Scheduled SLA Monitoring and Escalation

A separate scheduled path runs every hour. It pulls open Airtable tickets, loops through them one by one, checks whether the SLA deadline has passed, and sends an escalation message to FM management in Slack for breached tickets.

---

# 2. Block-by-Block Analysis

## 2.1 Complaint Ingestion

**Overview:**  
This block receives complaints from email or form submissions and converts both formats into a shared set of fields. That normalization allows the downstream AI and database steps to process either source identically.

**Nodes Involved:**  
- Gmail trigger
- Normalise email inputs
- Webhook
- Normalise form inputs
- Merge inputs

### Node: Gmail trigger
- **Type and role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger for incoming Gmail messages.
- **Configuration choices:**  
  - Polls every minute
  - Uses Gmail labels/filters: `UNREAD` and `INBOX`
  - `simple: false`, so richer Gmail payload data is available
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**  
  - Entry point
  - Outputs to **Normalise email inputs**
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases / failures:**  
  - Gmail OAuth credential failure
  - Polling quota or API throttling
  - Unexpected email shape, especially missing `from.value[0]`
  - HTML-only or unusual MIME body may affect `text`
- **Sub-workflow reference:** None

### Node: Normalise email inputs
- **Type and role:** `n8n-nodes-base.set`  
  Maps Gmail fields into a standardized complaint object.
- **Configuration choices:**  
  Creates these fields:
  - `source = "Email"`
  - `raw_complaint = {{$json.text}}`
  - `tenant_email = {{$json.from.value[0].address}}`
  - `tenant_name = {{$json.from.value[0].name}}`
  - `received_at = {{$json.date}}`
- **Key expressions or variables used:**  
  - `$json.text`
  - `$json.from.value[0].address`
  - `$json.from.value[0].name`
  - `$json.date`
- **Input and output connections:**  
  - Input from **Gmail trigger**
  - Output to **Merge inputs** on input 0
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**  
  - Empty email text
  - Sender array missing or malformed
  - Date field missing or unparsable
- **Sub-workflow reference:** None

### Node: Webhook
- **Type and role:** `n8n-nodes-base.webhook`  
  Accepts complaint submissions from external forms via HTTP POST.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path set to the workflow-specific webhook path
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point
  - Outputs to **Normalise form inputs**
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases / failures:**  
  - Incorrect payload shape from the form system
  - Form system not sending POST
  - Production vs test webhook URL confusion
- **Sub-workflow reference:** None

### Node: Normalise form inputs
- **Type and role:** `n8n-nodes-base.set`  
  Maps form payload fields into the same standardized complaint schema used by the email path.
- **Configuration choices:**  
  Creates:
  - `source = "Form"`
  - `raw_complaint = {{$json.body.data.fields[2].value}}`
  - `tenant_email = {{$json.body.data.fields[1].value}}`
  - `tenant_name = {{$json.body.data.fields[0].value}}`
  - `received_at = {{$json.body.createdAt}}`
- **Key expressions or variables used:**  
  - `$json.body.data.fields[2].value`
  - `$json.body.data.fields[1].value`
  - `$json.body.data.fields[0].value`
  - `$json.body.createdAt`
- **Input and output connections:**  
  - Input from **Webhook**
  - Output to **Merge inputs** on input 1
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**  
  - Hard-coded field indexes assume a specific form layout
  - Missing `body.data.fields`
  - Field order changes in the form will silently break data mapping
- **Sub-workflow reference:** None

### Node: Merge inputs
- **Type and role:** `n8n-nodes-base.merge`  
  Joins either normalized input stream into one downstream path.
- **Configuration choices:**  
  Default merge behavior; in this design it acts as a convergence point for the two source branches.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Inputs from **Normalise email inputs** and **Normalise form inputs**
  - Output to **LLM extract & classify**
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**  
  - Merge behavior depends on execution timing and mode defaults
  - If both branches were active simultaneously in unusual tests, behavior should be verified
- **Sub-workflow reference:** None

---

## 2.2 AI Triage and Ticket Enrichment

**Overview:**  
This block sends the complaint text to Claude with a strict structured-output prompt. The response is parsed, validated, and enriched with internal metadata such as ticket ID and SLA deadline.

**Nodes Involved:**  
- LLM extract & classify
- Parse LLM output

### Node: LLM extract & classify
- **Type and role:** `@n8n/n8n-nodes-langchain.anthropic`  
  Uses Anthropic Claude to classify the complaint and draft a tenant acknowledgement.
- **Configuration choices:**  
  - Model: `claude-sonnet-4-6`
  - Temperature: `0.1` for stable, deterministic output
  - Prompt instructs the model to:
    - normalize multilingual tenant text into English
    - return only valid JSON
    - use exact fields:
      - `unit`
      - `fault_category`
      - `urgency_indicators`
      - `tenant_tone`
      - `recurrence_signal`
      - `priority_tier`
      - `summary`
      - `ack_email_body`
    - follow explicit priority rules and fault-category rules
- **Key expressions or variables used:**  
  - User message content: `{{$json.raw_complaint}}`
- **Input and output connections:**  
  - Input from **Merge inputs**
  - Output to **Parse LLM output**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:**  
  - Anthropic credential/authentication failure
  - Model availability or naming mismatch
  - Non-JSON or partially formatted response despite instructions
  - Hallucinated category outside allowed values
- **Sub-workflow reference:** None

### Node: Parse LLM output
- **Type and role:** `n8n-nodes-base.code`  
  Parses the Claude response into structured fields, validates JSON, reattaches original complaint metadata, generates a ticket ID, and computes an SLA deadline.
- **Configuration choices:**  
  The JavaScript performs:
  1. extraction of the response content from multiple possible Anthropic output shapes
  2. removal of markdown code fences
  3. `JSON.parse()` validation
  4. retrieval of the original normalized complaint from `$('Merge inputs').first().json`
  5. SLA mapping:
     - `P1 -> 2`
     - `P2 -> 4`
     - `P3 -> 24`
  6. generation of `sla_deadline` using current execution time
  7. generation of a short random ticket ID such as `FM-ABC123`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `$('Merge inputs').first().json`
  - `Date.now()`
  - `Math.random().toString(36)...`
- **Input and output connections:**  
  - Input from **LLM extract & classify**
  - Output to **Search technician**
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**  
  - LLM returns invalid JSON
  - LLM output wrapped in unexpected structure not covered by the extraction logic
  - Missing `priority_tier` defaults SLA to 24 hours
  - Ticket ID generation is random and not collision-proof
- **Sub-workflow reference:** None

---

## 2.3 Technician Assignment and Work Order Creation

**Overview:**  
This block maps the AI-generated fault category to a technician record in Airtable, then creates a complaint/work-order record in Airtable with all complaint, SLA, and assignment data.

**Nodes Involved:**  
- Search technician
- Create work order

### Node: Search technician
- **Type and role:** `n8n-nodes-base.airtable`  
  Searches a technician table for the technician responsible for the classified fault category.
- **Configuration choices:**  
  - Operation: `search`
  - Returns a single match (`returnAll: false`)
  - Only fetches:
    - `Technician Name`
    - `Technician Email`
  - Filter formula:
    - `={Fault Category} = '{{ $json.fault_category }}'`
- **Key expressions or variables used:**  
  - `{{$json.fault_category}}`
- **Input and output connections:**  
  - Input from **Parse LLM output**
  - Output to **Create work order**
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases / failures:**  
  - Airtable credential failure
  - Base/table not selected yet in template
  - No technician found for a category
  - Formula quoting issues if values are unexpected
- **Sub-workflow reference:** None

### Node: Create work order
- **Type and role:** `n8n-nodes-base.airtable`  
  Creates a new record in the Airtable complaints table.
- **Configuration choices:**  
  - Operation: `create`
  - Mapping mode: define fields manually
  - Sets fields including:
    - `Unit`
    - `Source`
    - `Status = Open`
    - `Priority`
    - `Complaint`
    - `Recurrence`
    - `CreatedDate = {{$now}}`
    - `Tenant Name`
    - `Tenant Tone`
    - `SLA Deadline`
    - `Tenant Email`
    - `Ticket ID`
    - `Fault Category`
    - `Acknowledgement Sent = false`
    - `Assigned Technician Name`
    - `Assigned Technician Email`
  - Pulls values from both **Parse LLM output** and **Merge inputs**
- **Key expressions or variables used:**  
  Examples:
  - `{{$('Parse LLM output').item.json.unit}}`
  - `{{$('Merge inputs').item.json.source}}`
  - `{{$('Parse LLM output').item.json.priority_tier}}`
  - `{{$json['Technician Name']}}`
- **Input and output connections:**  
  - Input from **Search technician**
  - Output to **Send ACK email**
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases / failures:**  
  - Airtable schema mismatch
  - Field names must match exactly, including the `Ticket ID` field that appears to contain a hidden BOM/zero-width prefix in the schema
  - Option-type fields must contain allowed values only
  - If no technician was found upstream, assignment fields may be blank
- **Sub-workflow reference:** None

---

## 2.4 Tenant Acknowledgement and Team Dispatch

**Overview:**  
This block sends the tenant an acknowledgement email, updates the Airtable ticket to mark the acknowledgement as sent, and posts a Slack dispatch message containing the ticket summary and assignment details.

**Nodes Involved:**  
- Send ACK email
- Update ACK status
- Send a message

### Node: Send ACK email
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends a plain-text acknowledgement email to the tenant.
- **Configuration choices:**  
  - Recipient: `{{$('Merge inputs').item.json.tenant_email}}`
  - Subject format: `[TICKET_ID] Complaint received — FAULT_CATEGORY`
  - Message body uses the AI-generated acknowledgement plus:
    - ticket reference
    - expected response time in hours
    - localized SLA deadline in Singapore time
    - FM Management Team sign-off
  - Email type: `text`
- **Key expressions or variables used:**  
  - `{{$('Parse LLM output').item.json.ack_email_body}}`
  - `{{$('Parse LLM output').item.json.ticket_id}}`
  - `{{$('Parse LLM output').item.json.sla_hours}}`
  - `new Date(...).toLocaleString('en-SG', { timeZone: 'Asia/Singapore', ... })`
- **Input and output connections:**  
  - Input from **Create work order**
  - Output to **Update ACK status**
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases / failures:**  
  - Gmail send permission issues
  - Invalid tenant email
  - AI-generated email body could be empty if parsing was incomplete
  - Locale/timezone formatting depends on runtime support
- **Sub-workflow reference:** None

### Node: Update ACK status
- **Type and role:** `n8n-nodes-base.airtable`  
  Updates the just-created Airtable record to mark `Acknowledgement Sent` as true.
- **Configuration choices:**  
  - Operation: `update`
  - Matches on Airtable record `id`
  - Sets:
    - `id = {{$('Create work order').item.json.id}}`
    - `Acknowledgement Sent = true`
- **Key expressions or variables used:**  
  - `{{$('Create work order').item.json.id}}`
- **Input and output connections:**  
  - Input from **Send ACK email**
  - Output to **Send a message**
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases / failures:**  
  - Airtable record ID missing if create step failed or output changed
  - Schema drift between create table and update table configuration
- **Sub-workflow reference:** None

### Node: Send a message
- **Type and role:** `n8n-nodes-base.slack`  
  Posts a Slack notification for the newly created work order.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Sends a block message to a configured channel
  - Includes:
    - ticket ID
    - priority
    - fault category
    - unit
    - tenant name
    - summary
    - recurrence
    - assigned technician
    - SLA deadline in SGT
  - Also contains a plain text fallback field
- **Key expressions or variables used:**  
  - `{{$('Create work order').item.json.fields['﻿Ticket ID']}}`
  - `{{$('Parse LLM output').item.json.summary}}`
  - `{{$('Create work order').item.json.fields.Recurrence}}`
  - `new Date(...).toLocaleString('en-SG', ...)`
- **Input and output connections:**  
  - Input from **Update ACK status**
  - No downstream node
- **Version-specific requirements:** `typeVersion: 2.4`
- **Edge cases / failures:**  
  - Slack OAuth or channel access failure
  - The plain-text expression references `RecurrenceFlag`, which is not present in the created record; this likely produces wrong output in the fallback text
  - Field name `﻿Ticket ID` again appears to contain a hidden prefix and must match Airtable exactly
- **Sub-workflow reference:** None

---

## 2.5 Scheduled SLA Monitoring and Escalation

**Overview:**  
This independent branch checks all open tickets every hour. It loops through each ticket, compares the current time to the stored SLA deadline, and escalates overdue tickets to management in Slack.

**Nodes Involved:**  
- Schedule Trigger
- Search records
- Loop over tickets
- SLA breached?
- Escalate to FM Management

### Node: Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Launches the SLA-monitoring branch on a recurring schedule.
- **Configuration choices:**  
  - Interval rule configured on `hours`
  - Effectively runs hourly
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point
  - Outputs to **Search records**
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases / failures:**  
  - Workflow inactive
  - Timezone interpretation depends on n8n instance settings
- **Sub-workflow reference:** None

### Node: Search records
- **Type and role:** `n8n-nodes-base.airtable`  
  Searches the complaints table for all tickets not marked closed.
- **Configuration choices:**  
  - Operation: `search`
  - Filter formula: `NOT({Status} = 'Closed')`
  - `returnAll: false`
- **Key expressions or variables used:** None in formula beyond static Airtable expression.
- **Input and output connections:**  
  - Input from **Schedule Trigger**
  - Output to **Loop over tickets**
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases / failures:**  
  - `returnAll: false` means it may not fetch every open ticket if there are many records; this is important for SLA monitoring
  - Airtable credential or base/table configuration errors
- **Sub-workflow reference:** None

### Node: Loop over tickets
- **Type and role:** `n8n-nodes-base.splitInBatches`  
  Iterates through ticket records one at a time for SLA evaluation.
- **Configuration choices:**  
  - `reset: false`
  - Uses the loop output to continue processing until all fetched items are exhausted
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from **Search records**
  - Output 1 to **SLA breached?**
  - Loop-back input from **Escalate to FM Management**
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**  
  - If no items are returned, downstream steps do not run
  - Loop design is valid, but should be tested with non-breached items since only the true branch is wired from the IF node
- **Sub-workflow reference:** None

### Node: SLA breached?
- **Type and role:** `n8n-nodes-base.if`  
  Compares current time to a ticket’s SLA deadline.
- **Configuration choices:**  
  - DateTime condition: current time after `{{$json['SLA Deadline']}}`
  - Uses strict type validation
- **Key expressions or variables used:**  
  - `{{$now}}`
  - `{{$json['SLA Deadline']}}`
- **Input and output connections:**  
  - Input from **Loop over tickets**
  - True output to **Escalate to FM Management**
  - False output not connected
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases / failures:**  
  - The monitored Airtable records usually expose fields nested under `fields`, but this node references top-level `SLA Deadline`; whether this works depends on the exact Airtable output shape
  - Missing or invalid date causes expression/type errors or false negatives
  - Because the false path is not connected, loop continuation relies on whether the split/if behavior still allows all items to be processed as expected
- **Sub-workflow reference:** None

### Node: Escalate to FM Management
- **Type and role:** `n8n-nodes-base.slack`  
  Sends an urgent Slack alert for breached tickets.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Sends plain text to a configured channel
  - Includes:
    - ticket
    - priority
    - category
    - unit
    - tenant
    - assigned technician
    - status at deadline
    - operational instruction to update the ticket and notify tenant
- **Key expressions or variables used:**  
  - `{{$('Loop over tickets').item.json['﻿Ticket ID']}}`
  - `{{$('Loop over tickets').item.json.Priority}}`
  - `{{$('Loop over tickets').item.json['Fault Category']}}`
- **Input and output connections:**  
  - Input from **SLA breached?** true branch
  - Output loops back to **Loop over tickets**
- **Version-specific requirements:** `typeVersion: 2.4`
- **Edge cases / failures:**  
  - Slack permission/channel issues
  - Same possible Airtable field-shape mismatch as above if fields are nested
  - Message text mentions SharePoint, but the workflow uses Airtable; this is a documentation/configuration inconsistency
- **Sub-workflow reference:** None

---

## 2.6 Documentation / Annotation Nodes

**Overview:**  
These nodes do not execute business logic but provide in-canvas explanation, setup guidance, and customization notes. They are important for maintainability and should be preserved when reproducing the workflow.

**Nodes Involved:**  
- Sticky Note1
- Sticky Note
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node: Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Overall workflow description, setup requirements, and customization notes.
- **Configuration choices:** Large note covering the full process.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

### Node: Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Annotation for complaint ingestion.
- **Configuration choices:** Yellow note describing the two intake sources and normalized fields.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

### Node: Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Annotation for AI triage.
- **Configuration choices:** Yellow note describing Claude output and parsing behavior.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

### Node: Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Annotation for technician assignment and work-order creation.
- **Configuration choices:** Yellow note describing Airtable technician table requirements.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

### Node: Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Annotation for tenant acknowledgement and Slack dispatch.
- **Configuration choices:** Yellow note describing ACK email content and dispatch intent.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

### Node: Sticky Note5
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Annotation for hourly SLA monitoring.
- **Configuration choices:** Yellow note describing periodic open-ticket checks and escalation.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail trigger | Gmail Trigger | Poll shared inbox for new unread complaints |  | Normalise email inputs | **1. Complaint ingestion**<br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| Normalise email inputs | Set | Convert Gmail message into common complaint schema | Gmail trigger | Merge inputs | **1. Complaint ingestion**<br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| Webhook | Webhook | Receive complaint submissions from forms |  | Normalise form inputs | **1. Complaint ingestion**<br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| Normalise form inputs | Set | Convert form payload into common complaint schema | Webhook | Merge inputs | **1. Complaint ingestion**<br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| Merge inputs | Merge | Converge normalized complaint sources into one path | Normalise email inputs, Normalise form inputs | LLM extract & classify | **1. Complaint ingestion**<br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| LLM extract & classify | Anthropic (LangChain) | Extract structured complaint data and draft ACK email | Merge inputs | Parse LLM output | **2. AI triage (Claude Sonnet)**<br>Claude classifies the complaint and returns structured JSON:<br>- Fault category: ACMV \| Plumbing \| Electrical \| Cleanliness \| Access \| Noise<br>- Priority: P1 (2hr SLA) \| P2 (4hr SLA) \| P3 (24hr SLA)<br>- Tenant tone: Frustrated \| Neutral \| Urgent \| Abusive<br>- Recurrence signal, urgency indicators, plain-English summary<br>- Draft acknowledgement email (tone-matched, max 120 words)<br><br>The Parse LLM output node strips markdown fences, parses JSON, generates a ticket ID (FM-XXXXXX), and calculates the SLA deadline timestamp. |
| Parse LLM output | Code | Validate and parse AI output, add ticket ID and SLA deadline | LLM extract & classify | Search technician | **2. AI triage (Claude Sonnet)**<br>Claude classifies the complaint and returns structured JSON:<br>- Fault category: ACMV \| Plumbing \| Electrical \| Cleanliness \| Access \| Noise<br>- Priority: P1 (2hr SLA) \| P2 (4hr SLA) \| P3 (24hr SLA)<br>- Tenant tone: Frustrated \| Neutral \| Urgent \| Abusive<br>- Recurrence signal, urgency indicators, plain-English summary<br>- Draft acknowledgement email (tone-matched, max 120 words)<br><br>The Parse LLM output node strips markdown fences, parses JSON, generates a ticket ID (FM-XXXXXX), and calculates the SLA deadline timestamp. |
| Search technician | Airtable | Find technician by fault category | Parse LLM output | Create work order | **3. Technician assignment & work order**<br>Looks up the on-call technician for the fault category from your Airtable Technician table, then creates a full work order in the Complaints table.<br><br>The Technician table must have at least one row per fault category (ACMV, Plumbing, Electrical, Cleanliness, Access, Noise) with Technician Name and Technician Email columns. |
| Create work order | Airtable | Create complaint/work-order record in Airtable | Search technician | Send ACK email | **3. Technician assignment & work order**<br>Looks up the on-call technician for the fault category from your Airtable Technician table, then creates a full work order in the Complaints table.<br><br>The Technician table must have at least one row per fault category (ACMV, Plumbing, Electrical, Cleanliness, Access, Noise) with Technician Name and Technician Email columns. |
| Send ACK email | Gmail | Send acknowledgement email to tenant | Create work order | Update ACK status | **4. Tenant acknowledgement and dispatch**<br>Sends the AI-drafted acknowledgement email to the tenant with:<br>- Ticket reference (FM-XXXXXX)<br>- SLA commitment (e.g. "within 4 hours — by 3:45 PM SGT")<br><br>Acknowledgement Sent is then updated to true in Airtable so the SLA monitor knows not to double-escalate on a fresh ticket.<br><br>On-call technician is dispatched via an AI-drafted Slack message. |
| Update ACK status | Airtable | Mark acknowledgement as sent | Send ACK email | Send a message | **4. Tenant acknowledgement and dispatch**<br>Sends the AI-drafted acknowledgement email to the tenant with:<br>- Ticket reference (FM-XXXXXX)<br>- SLA commitment (e.g. "within 4 hours — by 3:45 PM SGT")<br><br>Acknowledgement Sent is then updated to true in Airtable so the SLA monitor knows not to double-escalate on a fresh ticket.<br><br>On-call technician is dispatched via an AI-drafted Slack message. |
| Send a message | Slack | Notify FM team in Slack about new work order | Update ACK status |  | **4. Tenant acknowledgement and dispatch**<br>Sends the AI-drafted acknowledgement email to the tenant with:<br>- Ticket reference (FM-XXXXXX)<br>- SLA commitment (e.g. "within 4 hours — by 3:45 PM SGT")<br><br>Acknowledgement Sent is then updated to true in Airtable so the SLA monitor knows not to double-escalate on a fresh ticket.<br><br>On-call technician is dispatched via an AI-drafted Slack message. |
| Schedule Trigger | Schedule Trigger | Launch hourly SLA monitoring |  | Search records | **5. Hourly SLA monitoring**<br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |
| Search records | Airtable | Search open complaint records | Schedule Trigger | Loop over tickets | **5. Hourly SLA monitoring**<br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |
| Loop over tickets | Split In Batches | Iterate through open tickets for SLA evaluation | Search records, Escalate to FM Management | SLA breached? | **5. Hourly SLA monitoring**<br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |
| SLA breached? | IF | Check whether current time is past SLA deadline | Loop over tickets | Escalate to FM Management | **5. Hourly SLA monitoring**<br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |
| Escalate to FM Management | Slack | Alert management about SLA-breached tickets | SLA breached? | Loop over tickets | **5. Hourly SLA monitoring**<br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |
| Sticky Note1 | Sticky Note | Overall workflow notes and setup instructions |  |  | ## FM Complaint Triage & SLA Escalation<br>Facility management teams waste hours manually reading tenant emails, deciding urgency, and chasing overdue tickets. This workflow automates the entire intake-to-dispatch loop from complaint received to technician notified.<br><br>### Who's it for<br>FM managers and operations teams in commercial buildings who receive tenant complaints by email or web form.<br><br>### How it works<br>1. Complaints arrive via Gmail or a web form webhook<br>2. Claude AI classifies each complaint: fault category, priority (P1/P2/P3), tenant tone, and drafts an acknowledgement email<br>3. The right technician is looked up in Airtable by fault category<br>4. A work order is created and the tenant receives an ACK email with their ticket reference and SLA commitment<br>5. The FM team is notified in Slack with ticket summary<br>6. An hourly schedule checks open tickets — any past their SLA deadline trigger an urgent escalation to FM management<br><br>### How to set up<br>1. Connect Gmail to the **Gmail Trigger** and **Send ACK email** nodes<br>2. Create your Airtable base with a **Complaints** table and a **Technician** table (one row per fault category)<br>3. Connect Airtable, Anthropic, and Slack in their respective nodes<br>4. If using a web form, point it to the Webhook URL<br><br>### Requirements<br>- Gmail account<br>- Airtable account<br>- Anthropic API key<br>- Slack workspace.<br><br>### How to customize the workflow<br>- Edit the LLM system prompt and the **Parse LLM output** node to change fault categories or SLA hours.<br>- Point the escalation node to a separate management-only Slack channel if needed.<br>- Remove the Webhook trigger and **Normalise form inputs** node if you only use email. |
| Sticky Note | Sticky Note | Annotation for complaint ingestion |  |  | **1. Complaint ingestion**<br><br>Accepts complaints from two sources:<br>- **Gmail Trigger** polls your shared FM inbox every minute for new INBOX messages<br>- **Webhook** receives POST requests from a web form (Tally, Typeform, Power Automate, etc.)<br><br>Both paths are normalised into identical fields (source, raw_complaint, tenant_email, tenant_name, received_at) and merged into a single item before classification. |
| Sticky Note2 | Sticky Note | Annotation for AI triage |  |  | **2. AI triage (Claude Sonnet)**<br><br>Claude classifies the complaint and returns structured JSON:<br>- Fault category: ACMV \| Plumbing \| Electrical \| Cleanliness \| Access \| Noise<br>- Priority: P1 (2hr SLA) \| P2 (4hr SLA) \| P3 (24hr SLA)<br>- Tenant tone: Frustrated \| Neutral \| Urgent \| Abusive<br>- Recurrence signal, urgency indicators, plain-English summary<br>- Draft acknowledgement email (tone-matched, max 120 words)<br><br>The Parse LLM output node strips markdown fences, parses JSON, generates a ticket ID (FM-XXXXXX), and calculates the SLA deadline timestamp. |
| Sticky Note3 | Sticky Note | Annotation for technician assignment |  |  | **3. Technician assignment & work order**<br><br>Looks up the on-call technician for the fault category from your Airtable Technician table, then creates a full work order in the Complaints table.<br><br>The Technician table must have at least one row per fault category (ACMV, Plumbing, Electrical, Cleanliness, Access, Noise) with Technician Name and Technician Email columns. |
| Sticky Note4 | Sticky Note | Annotation for ACK and dispatch |  |  | **4. Tenant acknowledgement and dispatch**<br><br>Sends the AI-drafted acknowledgement email to the tenant with:<br>- Ticket reference (FM-XXXXXX)<br>- SLA commitment (e.g. "within 4 hours — by 3:45 PM SGT")<br><br>Acknowledgement Sent is then updated to true in Airtable so the SLA monitor knows not to double-escalate on a fresh ticket.<br><br>On-call technician is dispatched via an AI-drafted Slack message. |
| Sticky Note5 | Sticky Note | Annotation for hourly SLA monitoring |  |  | **5. Hourly SLA monitoring**<br><br>Runs independently on a 1-hour schedule. Fetches all open tickets from Airtable, loops through each one, and checks whether the SLA Deadline has passed.<br><br>Any breached ticket triggers an urgent Slack message to the FM management channel with ticket details, assigned technician, and current status.<br><br>Tickets are only removed from the escalation loop once their Status is set to "Closed" in Airtable. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence.

## 4.1 Prepare external systems first

1. **Create or identify a Gmail account** for the FM inbox.
2. **Create an Anthropic API credential** for Claude access.
3. **Create a Slack app / OAuth connection** with permission to post to:
   - the operations channel
   - the management escalation channel
4. **Create an Airtable base** with at least two tables:

### Complaints table
Recommended fields:
- `Ticket ID` or exact imported equivalent
- `Tenant Name`
- `Tenant Email`
- `Unit`
- `Source` as single select: `Email`, `Form`
- `Fault Category` as single select: `Access`, `ACMV`, `Cleanliness`, `Electrical`, `Noise`, `Plumbing`
- `Complaint`
- `Priority` as single select: `P1`, `P2`, `P3`
- `Tenant Tone`
- `SLA Deadline` as date/time
- `Status` as single select: `Open`, `Closed`
- `Assigned Technician Name`
- `Assigned Technician Email`
- `Acknowledgement Sent` as checkbox/boolean
- `Closed Date` as date/time
- `CreatedDate` as date/time
- `Recurrence` as single select: `Yes`, `No`

### Technician table
Fields:
- `Fault Category`
- `Technician Name`
- `Technician Email`

Populate one row per category.

> Important: in the provided workflow, the Airtable field `Ticket ID` appears to include an invisible prefix character in some expressions. When rebuilding manually, use a clean field name like `Ticket ID` consistently and update all expressions accordingly.

## 4.2 Build the complaint intake path

5. Add a **Gmail Trigger** node named **Gmail trigger**.
   - Poll mode: every minute
   - Filters: labels `UNREAD` and `INBOX`
   - Set `Simple` to `false`
   - Attach Gmail credentials

6. Add a **Set** node named **Normalise email inputs**.
   - Add fields:
     - `source` = `Email`
     - `raw_complaint` = `{{$json.text}}`
     - `tenant_email` = `{{$json.from.value[0].address}}`
     - `tenant_name` = `{{$json.from.value[0].name}}`
     - `received_at` = `{{$json.date}}`
   - Connect **Gmail trigger -> Normalise email inputs**

7. Add a **Webhook** node named **Webhook**.
   - HTTP Method: `POST`
   - Choose a path
   - Save the production URL for your form tool

8. Add a **Set** node named **Normalise form inputs**.
   - Add fields:
     - `source` = `Form`
     - `raw_complaint` = `{{$json.body.data.fields[2].value}}`
     - `tenant_email` = `{{$json.body.data.fields[1].value}}`
     - `tenant_name` = `{{$json.body.data.fields[0].value}}`
     - `received_at` = `{{$json.body.createdAt}}`
   - Connect **Webhook -> Normalise form inputs**

9. Add a **Merge** node named **Merge inputs**.
   - Keep default mode unless you deliberately choose a different convergence strategy
   - Connect:
     - **Normalise email inputs -> Merge inputs** input 1
     - **Normalise form inputs -> Merge inputs** input 2

> Recommended hardening: if you know your form payload may change, replace indexed form expressions with named-field extraction in a Code node.

## 4.3 Build the AI triage path

10. Add an **Anthropic** node from LangChain named **LLM extract & classify**.
   - Model: `claude-sonnet-4-6`
   - Temperature: `0.1`
   - Add the instruction prompt as the system/assistant message:

   Use the exact logic from the workflow:
   - multilingual tenant text accepted, but output must be English
   - JSON only
   - fields:
     - `unit`
     - `fault_category`
     - `urgency_indicators`
     - `tenant_tone`
     - `recurrence_signal`
     - `priority_tier`
     - `summary`
     - `ack_email_body`
   - categories limited to:
     - `ACMV`
     - `Plumbing`
     - `Electrical`
     - `Cleanliness`
     - `Access`
     - `Noise`
   - priority rules:
     - `P1` = 2 hr SLA
     - `P2` = 4 hr SLA
     - `P3` = 24 hr SLA
   - acknowledgement email:
     - professional
     - salutation, blank line, body
     - max 120 words
     - no sign-off

   - Add a user message:
     - `{{$json.raw_complaint}}`

11. Connect **Merge inputs -> LLM extract & classify**.

12. Add a **Code** node named **Parse LLM output** and paste logic equivalent to this behavior:
   - extract text from possible Anthropic response properties
   - strip markdown code fences
   - parse JSON
   - read original normalized complaint from **Merge inputs**
   - map priority to hours: `P1=2`, `P2=4`, `P3=24`
   - compute `sla_deadline`
   - create random ticket ID like `FM-ABC123`
   - output a single object combining:
     - original normalized fields
     - parsed AI fields
     - `ticket_id`
     - `sla_deadline`
     - `sla_hours`

13. Connect **LLM extract & classify -> Parse LLM output**.

## 4.4 Build technician lookup and ticket creation

14. Add an **Airtable** node named **Search technician**.
   - Operation: `Search`
   - Table: `Technician`
   - Return all: `false`
   - Optional fields to return:
     - `Technician Name`
     - `Technician Email`
   - Filter formula:
     - `={Fault Category} = '{{ $json.fault_category }}'`
   - Attach Airtable credentials

15. Connect **Parse LLM output -> Search technician**.

16. Add an **Airtable** node named **Create work order**.
   - Operation: `Create`
   - Table: `Complaints`
   - Map these fields:
     - `Unit` = `{{$('Parse LLM output').item.json.unit}}`
     - `Source` = `{{$('Merge inputs').item.json.source}}`
     - `Status` = `Open`
     - `Priority` = `{{$('Parse LLM output').item.json.priority_tier}}`
     - `Complaint` = `{{$('Parse LLM output').item.json.raw_complaint}}`
     - `Recurrence` = `{{$('Parse LLM output').item.json.recurrence_signal}}`
     - `CreatedDate` = `{{$now}}`
     - `Tenant Name` = `{{$('Parse LLM output').item.json.tenant_name}}`
     - `Tenant Tone` = `{{$('Parse LLM output').item.json.tenant_tone}}`
     - `SLA Deadline` = `{{$('Parse LLM output').item.json.sla_deadline}}`
     - `Tenant Email` = `{{$('Parse LLM output').item.json.tenant_email}}`
     - `Ticket ID` = `{{$('Parse LLM output').item.json.ticket_id}}`
     - `Fault Category` = `{{$('Parse LLM output').item.json.fault_category}}`
     - `Acknowledgement Sent` = `false`
     - `Assigned Technician Name` = `{{$json['Technician Name']}}`
     - `Assigned Technician Email` = `{{$json['Technician Email']}}`

17. Connect **Search technician -> Create work order**.

## 4.5 Build acknowledgement email and Slack dispatch

18. Add a **Gmail** node named **Send ACK email**.
   - Operation: send email
   - To:
     - `{{$('Merge inputs').item.json.tenant_email}}`
   - Subject:
     - `{{ '[' + $('Parse LLM output').item.json.ticket_id + '] Complaint received — ' + $('Parse LLM output').item.json.fault_category }}`
   - Email type: plain text
   - Body:
     - start with `{{$('Parse LLM output').item.json.ack_email_body}}`
     - append:
       - `Ticket reference: ...`
       - `Expected response: within X hour(s) — by localized SGT time`
       - `FM Management Team`

19. Connect **Create work order -> Send ACK email**.

20. Add an **Airtable** node named **Update ACK status**.
   - Operation: `Update`
   - Table: `Complaints`
   - Match by record `id`
   - Map:
     - `id` = `{{$('Create work order').item.json.id}}`
     - `Acknowledgement Sent` = `true`

21. Connect **Send ACK email -> Update ACK status**.

22. Add a **Slack** node named **Send a message**.
   - Authentication: OAuth2
   - Select the operations/dispatch channel
   - Message type: Blocks
   - Include:
     - ticket ID
     - priority
     - fault category
     - unit
     - tenant
     - summary
     - recurrence
     - assigned technician
     - SLA deadline in `Asia/Singapore`
   - Also add plain text fallback

23. Connect **Update ACK status -> Send a message**.

> Recommended fix while rebuilding: in the text fallback, use `Recurrence` instead of the broken `RecurrenceFlag` expression from the original workflow.

## 4.6 Build the SLA monitoring branch

24. Add a **Schedule Trigger** node named **Schedule Trigger**.
   - Run every 1 hour

25. Add an **Airtable** node named **Search records**.
   - Operation: `Search`
   - Table: `Complaints`
   - Filter formula:
     - `NOT({Status} = 'Closed')`
   - Prefer `returnAll = true` when rebuilding, so all open records are checked

26. Connect **Schedule Trigger -> Search records**.

27. Add a **Split In Batches** node named **Loop over tickets**.
   - Keep default batch size of 1 if you want one-by-one checks
   - `reset: false`

28. Connect **Search records -> Loop over tickets**.

29. Add an **IF** node named **SLA breached?**.
   - Condition type: Date & Time
   - Compare:
     - left = `{{$now}}`
     - operation = `after`
     - right = the ticket deadline field

   If Airtable outputs data under `fields`, use:
   - `{{$json.fields['SLA Deadline']}}`

   If your Airtable node flattens fields, use:
   - `{{$json['SLA Deadline']}}`

30. Connect **Loop over tickets -> SLA breached?**.

31. Add a **Slack** node named **Escalate to FM Management**.
   - Authentication: OAuth2
   - Select management-only Slack channel
   - Message text should include:
     - `SLA BREACH — Immediate Action Required`
     - ticket
     - priority
     - category
     - unit
     - tenant
     - assigned technician
     - current status
     - action instruction

32. Connect the **true** output of **SLA breached? -> Escalate to FM Management**.

33. Connect **Escalate to FM Management -> Loop over tickets** to continue the batch loop.

> Important rebuild note: the original workflow only loops back after escalation. To make looping fully robust, verify behavior for both breached and non-breached tickets. In some builds, you may prefer a different loop pattern or explicitly connect a no-op path for the false branch.

## 4.7 Add documentation notes

34. Add sticky notes for:
   - overall workflow summary and setup
   - complaint ingestion
   - AI triage
   - technician assignment
   - tenant acknowledgement and dispatch
   - hourly SLA monitoring

This is optional for execution but useful for maintainability.

## 4.8 Configure credentials

35. **Gmail credentials**
   - Attach to:
     - **Gmail trigger**
     - **Send ACK email**

36. **Anthropic credentials**
   - Attach to:
     - **LLM extract & classify**

37. **Airtable credentials**
   - Attach to:
     - **Search technician**
     - **Create work order**
     - **Update ACK status**
     - **Search records**

38. **Slack OAuth2 credentials**
   - Attach to:
     - **Send a message**
     - **Escalate to FM Management**

## 4.9 Test sequence

39. Test with:
   - one sample Gmail complaint
   - one sample webhook complaint
   - one synthetic P1 complaint
   - one unknown-category complaint
   - one complaint with no unit number

40. Verify:
   - LLM returns valid JSON
   - technician lookup succeeds
   - Airtable record is created
   - ACK email sends
   - Slack dispatch posts
   - hourly branch escalates breached tickets

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| FM Complaint Triage & SLA Escalation: automates intake-to-dispatch from complaint received to technician notified. | Overall workflow purpose |
| Intended for FM managers and operations teams in commercial buildings receiving tenant complaints by email or web form. | Audience |
| Setup summary: connect Gmail, Airtable, Anthropic, and Slack; point the form system to the webhook URL if forms are used. | Deployment guidance |
| Requirements: Gmail account, Airtable account, Anthropic API key, Slack workspace. | Prerequisites |
| Customization options: edit the LLM prompt and Parse LLM output node to change categories or SLA hours. | Maintenance guidance |
| Customization options: point escalation to a separate management-only Slack channel if needed. | Operations guidance |
| Customization options: remove Webhook and Normalise form inputs if only email intake is required. | Simplification guidance |

## Additional implementation observations

- The workflow has **multiple entry points**:
  - **Gmail trigger**
  - **Webhook**
  - **Schedule Trigger**
- The workflow uses **no sub-workflows**.
- There are a few important inconsistencies in the current JSON that should be corrected in production:
  1. **Search records** uses `returnAll: false`, which may miss open tickets during SLA monitoring.
  2. **Escalate to FM Management** says “update the ticket in SharePoint,” but the workflow uses Airtable.
  3. **Send a message** plain-text fallback references `RecurrenceFlag`, which is not created anywhere.
  4. Airtable field naming for **Ticket ID** appears to include an invisible character in some mappings; standardize this when rebuilding.
  5. The SLA check may need to reference `fields['SLA Deadline']` depending on your Airtable node output structure.