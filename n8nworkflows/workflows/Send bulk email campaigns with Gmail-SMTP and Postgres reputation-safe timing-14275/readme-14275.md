Send bulk email campaigns with Gmail/SMTP and Postgres reputation-safe timing

https://n8nworkflows.xyz/workflows/send-bulk-email-campaigns-with-gmail-smtp-and-postgres-reputation-safe-timing-14275


# Send bulk email campaigns with Gmail/SMTP and Postgres reputation-safe timing

# 1. Workflow Overview

This workflow implements a bulk email campaign system in n8n with three major capabilities:

- **Lead intake and validation** from an uploaded CSV via webhook
- **Scheduled campaign execution** with time-zone-aware send timing and inbox throttling
- **Event tracking and analytics feedback loop** to improve future send-time rules

Its intended use cases include outbound campaign operations where deliverability matters, inbox reputation must be protected, and send timing should gradually improve from historical engagement data.

## 1.1 Lead Intake and Campaign Setup

A webhook receives campaign data and a CSV file of leads. The workflow then applies central configuration values, extracts rows from the uploaded file, validates email syntax, and separates valid from invalid leads.

## 1.2 Lead Validation and Enrichment

Valid leads are checked against MX records, enriched with domain classification, timezone, and a heuristic engagement score, then inserted into Postgres. Invalid leads are stored separately for auditing or cleanup.

## 1.3 Scheduled Campaign Execution

A scheduler periodically looks for active campaigns that are ready to run. For each campaign, it fetches the top pending leads, calculates an optimal send time per lead, selects an inbox under hourly/daily limits, waits until the target time, and sends the email.

## 1.4 Delivery Logging and Inbox Usage Tracking

After each email is sent, the workflow records the send event and updates inbox statistics so future sends can respect throughput limits.

## 1.5 Email Event Intake and Performance Updates

A second webhook accepts downstream email events such as opens, clicks, replies, and bounces. These events are normalized, logged, and used to update campaign-level performance counters.

## 1.6 Periodic Analytics and Continuous Optimization

A separate scheduler periodically analyzes historical email events by timezone and hour. It then updates the `send_time_rules` table so future campaign sends can use better-performing send windows.

---

# 2. Block-by-Block Analysis

## Block 1 — Campaign Upload, Configuration, and Lead Extraction

### Overview
This block receives inbound campaign submissions and parses the uploaded CSV file into one item per lead. It is the entry point for campaign data ingestion.

### Nodes Involved
- Campaign Upload Webhook
- Workflow Configuration
- Extract CSV Leads
- Split Leads

### Node Details

#### 1. Campaign Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that accepts an HTTP POST request for campaign uploads.
- **Configuration choices:**  
  - Path: `campaign-upload`
  - Method: `POST`
  - Response mode: `lastNode`
- **Key expressions or variables used:**  
  None directly in parameters.
- **Input and output connections:**  
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:**  
  Uses webhook node type version `2.1`.
- **Edge cases or potential failure types:**  
  - Wrong HTTP method
  - Missing multipart/binary file data for CSV extraction downstream
  - Long-running request because response is sent only after the last node in this branch
- **Sub-workflow reference:**  
  None

#### 2. Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects reusable workflow constants into the item stream.
- **Configuration choices:**  
  Adds:
  - `maxEmailsPerInboxPerHour = 50`
  - `maxEmailsPerInboxPerDay = 500`
  - `minDelayBetweenEmailsMinutes = 2`
  - `mxRecordCheckApiUrl = https://dns.google/resolve`
  - `defaultTimezone = UTC`
  Keeps original fields with `includeOtherFields = true`.
- **Key expressions or variables used:**  
  These values are later referenced, especially `mxRecordCheckApiUrl`.
- **Input and output connections:**  
  - Input: `Campaign Upload Webhook`
  - Output: `Extract CSV Leads`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Configuration values exist only in this ingestion branch; they are not automatically available in scheduler branches unless persisted externally
- **Sub-workflow reference:**  
  None

#### 3. Extract CSV Leads
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Parses an uploaded file into structured data.
- **Configuration choices:**  
  Options are left default. The node is expected to infer/parse the file content.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Input: `Workflow Configuration`
  - Output: `Split Leads`
- **Version-specific requirements:**  
  Version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing binary file input
  - Wrong file format
  - CSV delimiter/header mismatch
  - Large file memory handling
- **Sub-workflow reference:**  
  None

#### 4. Split Leads
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits an array field into one item per lead.
- **Configuration choices:**  
  - Field to split out: `data`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Extract CSV Leads`
  - Output: `Validate Email & Extract Domain`
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases or potential failure types:**  
  - `data` missing because CSV parse structure differs
  - Empty CSV
- **Sub-workflow reference:**  
  None

---

## Block 2 — Email Validation and Routing

### Overview
This block validates lead email addresses and routes valid records toward enrichment while invalid ones are stored separately.

### Nodes Involved
- Validate Email & Extract Domain
- Is Email Valid?
- Store Invalid Lead

### Node Details

#### 5. Validate Email & Extract Domain
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs syntax validation using JavaScript and extracts the email domain.
- **Configuration choices:**  
  - Mode: `runOnceForEachItem`
  - Uses a simplified RFC 5322-style regex
  - Outputs original fields plus:
    - `isValid`
    - `domain`
    - `validatedAt`
- **Key expressions or variables used:**  
  - Reads `$input.item.json.email`
- **Input and output connections:**  
  - Input: `Split Leads`
  - Output: `Is Email Valid?`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - `email` missing or null
  - Unexpected types
  - Regex passes some addresses that are still not deliverable
- **Sub-workflow reference:**  
  None

#### 6. Is Email Valid?
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes items based on the boolean `isValid`.
- **Configuration choices:**  
  Condition checks whether `$('Validate Email & Extract Domain').item.json.isValid` is true.
- **Key expressions or variables used:**  
  - `={{ $('Validate Email & Extract Domain').item.json.isValid }}`
- **Input and output connections:**  
  - Input: `Validate Email & Extract Domain`
  - True output: `Check Domain MX Records`
  - False output: `Store Invalid Lead`
- **Version-specific requirements:**  
  IF node version `2.3`.
- **Edge cases or potential failure types:**  
  - Expression resolution issues if referenced node/item context changes
  - Loose type validation may coerce non-boolean values
- **Sub-workflow reference:**  
  None

#### 7. Store Invalid Lead
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts rejected leads into a separate table.
- **Configuration choices:**  
  - Schema: `public`
  - Table: `invalid_leads`
  - Columns:
    - `email = {{$json.email}}`
    - `reason = invalid_syntax_or_domain`
    - `campaign_id = {{$json.campaign_id}}`
- **Key expressions or variables used:**  
  - `{{$json.email}}`
  - `{{$json.campaign_id}}`
- **Input and output connections:**  
  - Input: false branch of `Is Email Valid?`
  - Output: none
- **Version-specific requirements:**  
  Postgres node version `2.6`.
- **Edge cases or potential failure types:**  
  - Database credential/authentication errors
  - Missing `campaign_id`
  - Table schema mismatch
- **Sub-workflow reference:**  
  None

---

## Block 3 — MX Verification, Lead Enrichment, and Storage

### Overview
This block checks whether the domain appears to have MX records, enriches lead metadata, and stores accepted leads in Postgres.

### Nodes Involved
- Check Domain MX Records
- Enrich Lead Data
- Store Valid Lead

### Node Details

#### 8. Check Domain MX Records
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Google DNS-over-HTTPS for MX record resolution.
- **Configuration choices:**  
  - URL built from workflow configuration:
    `{{ $('Workflow Configuration').first().json.mxRecordCheckApiUrl }}?name={{ $json.domain }}&type=MX`
  - Response format: JSON
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').first().json.mxRecordCheckApiUrl`
  - `$json.domain`
- **Input and output connections:**  
  - Input: true branch of `Is Email Valid?`
  - Output: `Enrich Lead Data`
- **Version-specific requirements:**  
  HTTP Request version `4.3`.
- **Edge cases or potential failure types:**  
  - DNS API unavailable or rate-limited
  - `domain` missing
  - Response structure not merged as expected with prior lead fields
- **Sub-workflow reference:**  
  None

#### 9. Enrich Lead Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Adds derived attributes such as `domainType`, `timezone`, `engagementScore`, and flags.
- **Configuration choices:**  
  - Mode: `runOnceForEachItem`
  - Domain classification:
    - `corporate`
    - `gmail`
    - `outlook`
    - `free`
  - Timezone derived from TLD with fallback to `UTC`
  - Engagement score begins at 50 and is increased heuristically
- **Key expressions or variables used:**  
  Reads `item.email`, `item.mxValid`, `item.hasMxRecords`
- **Input and output connections:**  
  - Input: `Check Domain MX Records`
  - Output: `Store Valid Lead`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - The previous HTTP node may overwrite or not preserve original lead fields depending on configuration and response shape
  - Output names use camelCase (`domainType`, `engagementScore`) but downstream DB insert expects snake_case-like references (`domain_type`, `engagement_score`)
  - TLD-to-timezone mapping is intentionally simplistic
- **Sub-workflow reference:**  
  None

#### 10. Store Valid Lead
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Persists accepted/enriched leads into the `leads` table.
- **Configuration choices:**  
  - Schema: `public`
  - Table: `leads`
  - Inserts:
    - `email`
    - `domain`
    - `status = pending`
    - `timezone`
    - `campaign_id`
    - `domain_type = {{$json.domain_type}}`
    - `engagement_score = {{$json.engagement_score}}`
- **Key expressions or variables used:**  
  - `{{$json.email}}`
  - `{{$json.domain}}`
  - `{{$json.timezone}}`
  - `{{$json.campaign_id}}`
  - `{{$json.domain_type}}`
  - `{{$json.engagement_score}}`
- **Input and output connections:**  
  - Input: `Enrich Lead Data`
  - Output: none
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Likely field mismatch: upstream code returns `domainType` and `engagementScore`, not `domain_type` and `engagement_score`
  - Missing campaign ID
  - DB constraints or required columns
- **Sub-workflow reference:**  
  None

---

## Block 4 — Campaign Scheduling and Lead Selection

### Overview
This block runs periodically to find campaigns ready for sending and retrieve a batch of the most promising pending leads.

### Nodes Involved
- Campaign Scheduler
- Get Pending Campaigns
- Split Campaigns
- Get Campaign Leads
- Split Campaign Leads

### Node Details

#### 11. Campaign Scheduler
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts campaign processing on a repeating schedule.
- **Configuration choices:**  
  Interval is configured on the `minutes` field, implying a periodic minute-based trigger.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none
  - Output: `Get Pending Campaigns`
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases or potential failure types:**  
  - Frequency may be too aggressive for long wait-heavy execution patterns
- **Sub-workflow reference:**  
  None

#### 12. Get Pending Campaigns
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Fetches campaigns that are active and due.
- **Configuration choices:**  
  Executes:
  `SELECT * FROM campaigns WHERE status = 'active' AND scheduled_time <= NOW()`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Campaign Scheduler`
  - Output: `Split Campaigns`
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Empty result set
  - Missing indexes on `status` or `scheduled_time`
- **Sub-workflow reference:**  
  None

#### 13. Split Campaigns
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits campaign query results into one item per campaign.
- **Configuration choices:**  
  - Field to split out: `data`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Get Pending Campaigns`
  - Output: `Get Campaign Leads`
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases or potential failure types:**  
  - `data` missing if DB node output mode changes
- **Sub-workflow reference:**  
  None

#### 14. Get Campaign Leads
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Retrieves a prioritized batch of pending leads for the current campaign.
- **Configuration choices:**  
  Executes:
  `SELECT l.*, st.best_send_hour FROM leads l LEFT JOIN send_time_rules st ON l.timezone = st.timezone WHERE l.campaign_id = {{ $json.id }} AND l.status = 'pending' ORDER BY l.engagement_score DESC LIMIT 100`
- **Key expressions or variables used:**  
  - `{{$json.id}}`
- **Input and output connections:**  
  - Input: `Split Campaigns`
  - Output: `Split Campaign Leads`
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - SQL interpolation rather than parameterized query
  - `id` missing
  - `best_send_hour` null if no send-time rule exists
  - No inbox data is fetched here even though later logic expects available inboxes
- **Sub-workflow reference:**  
  None

#### 15. Split Campaign Leads
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the selected leads into one item per lead.
- **Configuration choices:**  
  - Field to split out: `data`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Get Campaign Leads`
  - Output: `Calculate Send Time & Select Inbox`
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases or potential failure types:**  
  - Empty lead batch
  - `data` structure mismatch
- **Sub-workflow reference:**  
  None

---

## Block 5 — Send-Time Calculation, Inbox Selection, and Delivery

### Overview
This block computes the send timestamp for each lead, picks an eligible inbox according to throttle rules, waits until the calculated time, then sends the email.

### Nodes Involved
- Calculate Send Time & Select Inbox
- Wait Until Send Time
- Send Email

### Node Details

#### 16. Calculate Send Time & Select Inbox
- **Type and technical role:** `n8n-nodes-base.code`  
  Determines when to send and which inbox to use.
- **Configuration choices:**  
  - Mode: `runOnceForEachItem`
  - Uses lead timezone and `best_send_hour`
  - Attempts to use `luxon` for timezone-safe time calculation
  - Expects `available_inboxes` array on the incoming item
  - Selects first inbox under daily/hourly thresholds
- **Key expressions or variables used:**  
  Reads:
  - `item.timezone`
  - `item.best_send_hour`
  - `item.campaign_id`
  - `item.available_inboxes`
- **Input and output connections:**  
  - Input: `Split Campaign Leads`
  - Output: `Wait Until Send Time`
- **Version-specific requirements:**  
  Code node version `2`.  
  Use of `require('luxon')` depends on n8n runtime support for external modules in Code nodes.
- **Edge cases or potential failure types:**  
  - `available_inboxes` is not populated anywhere upstream in this workflow, so this node will usually throw `No available inboxes found for campaign`
  - Runtime may not allow `require('luxon')`
  - Invalid timezone string
  - All inboxes at limits
- **Sub-workflow reference:**  
  None

#### 17. Wait Until Send Time
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution until the computed timestamp.
- **Configuration choices:**  
  - Resume mode: `specificTime`
  - Resume datetime: `{{$json.sendTime}}`
- **Key expressions or variables used:**  
  - `{{$json.sendTime}}`
- **Input and output connections:**  
  - Input: `Calculate Send Time & Select Inbox`
  - Output: `Send Email`
- **Version-specific requirements:**  
  Version `1.1`.
- **Edge cases or potential failure types:**  
  - Invalid or past timestamp
  - Large volume of waiting executions may stress n8n instance capacity
- **Sub-workflow reference:**  
  None

#### 18. Send Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the outbound email.
- **Configuration choices:**  
  - To: `{{$json.email}}`
  - Subject: `{{$json.subject}}`
  - Message: `{{$json.emailBody}}`
  - Sender name option: `{{$json.selectedInbox}}`
- **Key expressions or variables used:**  
  - `{{$json.email}}`
  - `{{$json.subject}}`
  - `{{$json.emailBody}}`
  - `{{$json.selectedInbox}}`
- **Input and output connections:**  
  - Input: `Wait Until Send Time`
  - Output: `Log Send Event`
- **Version-specific requirements:**  
  Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth not configured
  - `selectedInbox` is an object, but sender name expects a string; also Gmail node does not dynamically switch sending account based on item data
  - Missing subject/body fields; these are not created in this workflow
  - Rate limits and Gmail daily quotas
- **Sub-workflow reference:**  
  None

---

## Block 6 — Delivery Logging and Inbox Stats

### Overview
This block records successful send events and increments per-inbox counters used for reputation-safe throttling.

### Nodes Involved
- Log Send Event
- Update Inbox Stats

### Node Details

#### 19. Log Send Event
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts a “sent” event into `email_events`.
- **Configuration choices:**  
  Stores:
  - `lead_id`
  - `sent_at = {{$now.toISO()}}`
  - `event_type = sent`
  - `inbox_used`
  - `campaign_id`
- **Key expressions or variables used:**  
  - `{{$json.lead_id}}`
  - `{{$now.toISO()}}`
  - `{{$json.inbox_used}}`
  - `{{$json.campaign_id}}`
- **Input and output connections:**  
  - Input: `Send Email`
  - Output: `Update Inbox Stats`
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Incoming item may not include `lead_id` or `inbox_used`
  - Data model inconsistency with upstream `selectedInbox`
- **Sub-workflow reference:**  
  None

#### 20. Update Inbox Stats
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Updates the usage counters for the sending inbox.
- **Configuration choices:**  
  Executes:
  `UPDATE inbox_stats SET emails_sent_today = emails_sent_today + 1, emails_sent_this_hour = emails_sent_this_hour + 1, last_send_time = NOW() WHERE inbox_email = '={{ $json.selectedInbox }}'`
- **Key expressions or variables used:**  
  - `{{$json.selectedInbox}}`
- **Input and output connections:**  
  - Input: `Log Send Event`
  - Output: none
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - `selectedInbox` is an object upstream, but query compares against a string email column
  - SQL string construction appears malformed with embedded quoting
  - No row updated if inbox email does not match exactly
- **Sub-workflow reference:**  
  None

---

## Block 7 — Email Event Intake and Campaign Metrics Update

### Overview
This block accepts asynchronous email activity events and translates them into a consistent internal format for storage and metrics updates.

### Nodes Involved
- Email Event Webhook
- Parse Event Data
- Log Email Event
- Update Performance Stats

### Node Details

#### 21. Email Event Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for email tracking events such as opens and clicks.
- **Configuration choices:**  
  - Path: `email-events`
  - Method: `POST`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none
  - Output: `Parse Event Data`
- **Version-specific requirements:**  
  Webhook version `2.1`.
- **Edge cases or potential failure types:**  
  - Provider payload shape may differ
  - Authentication/signature verification is not implemented
- **Sub-workflow reference:**  
  None

#### 22. Parse Event Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes different incoming payload formats into a common schema.
- **Configuration choices:**  
  - Mode: `runOnceForEachItem`
  - Produces:
    - `eventType`
    - `emailIdentifier`
    - `timestamp`
    - `recipientEmail`
    - `campaignId`
    - `leadId`
    - `eventDetails`
    - `rawPayload`
- **Key expressions or variables used:**  
  Reads many alternative payload keys like `event_type`, `eventType`, `message_id`, `recipient`, etc.
- **Input and output connections:**  
  - Input: `Email Event Webhook`
  - Output: `Log Email Event`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - Event payloads may still omit campaign/lead correlation data
  - Case normalization may hide provider-specific distinctions
- **Sub-workflow reference:**  
  None

#### 23. Log Email Event
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Stores normalized events in `email_events`.
- **Configuration choices:**  
  Inserts:
  - `lead_id = {{$json.lead_id}}`
  - `event_type = {{$json.event_type}}`
  - `inbox_used = {{$json.inbox_used}}`
  - `campaign_id = {{$json.campaign_id}}`
  - `event_timestamp = {{$json.event_timestamp}}`
- **Key expressions or variables used:**  
  Uses snake_case fields.
- **Input and output connections:**  
  - Input: `Parse Event Data`
  - Output: `Update Performance Stats`
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Major field mismatch: parser outputs `eventType`, `campaignId`, `leadId`, `timestamp`, but this node expects `event_type`, `campaign_id`, `lead_id`, `event_timestamp`
  - Missing inbox value
  - Duplicate insert risk depending on table constraints
- **Sub-workflow reference:**  
  None

#### 24. Update Performance Stats
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Updates aggregate counters in `campaign_stats`.
- **Configuration choices:**  
  SQL increments opens/clicks/replies based on `{{$json.event_type}}` for the given campaign.
- **Key expressions or variables used:**  
  - `{{$json.event_type}}`
  - `{{$json.campaign_id}}`
- **Input and output connections:**  
  - Input: `Log Email Event`
  - Output: none
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Same snake_case mismatch as above
  - SQL interpolation risks and quoting brittleness
  - Bounce events are ignored in aggregate stats
- **Sub-workflow reference:**  
  None

---

## Block 8 — Analytics Trigger and Send-Time Rule Optimization

### Overview
This block periodically analyzes historical open data and refreshes the `send_time_rules` table so the campaign scheduler can use better send-hour recommendations.

### Nodes Involved
- Analytics Scheduler
- Get Performance Data
- Calculate Best Send Times
- Update Send Time Rules

### Node Details

#### 25. Analytics Scheduler
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the periodic analytics cycle.
- **Configuration choices:**  
  Interval is present but effectively unspecified in the JSON (`{}`), so this requires validation in the UI.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: none
  - Output: `Get Performance Data`
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases or potential failure types:**  
  - Trigger may be invalid or default unexpectedly due to incomplete interval config
- **Sub-workflow reference:**  
  None

#### 26. Get Performance Data
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Pulls historical open-event data grouped by hour and timezone.
- **Configuration choices:**  
  Executes:
  `SELECT EXTRACT(HOUR FROM event_timestamp) as hour, l.timezone, COUNT(*) as open_count FROM email_events e JOIN leads l ON e.lead_id = l.id WHERE e.event_type = 'open' AND e.event_timestamp > NOW() - INTERVAL '30 days' GROUP BY hour, l.timezone ORDER BY l.timezone, open_count DESC`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: `Analytics Scheduler`
  - Output: `Calculate Best Send Times`
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Query output columns do not match what the next Code node expects
  - Limited metric depth: only opens, not sent counts or open rates
- **Sub-workflow reference:**  
  None

#### 27. Calculate Best Send Times
- **Type and technical role:** `n8n-nodes-base.code`  
  Attempts to compute the best send hour per timezone from analytics data.
- **Configuration choices:**  
  - Reads all input items
  - Groups by timezone and hour
  - Expects fields:
    - `send_hour`
    - `open_rate`
    - `total_sent`
    - `total_opened`
  - Outputs:
    - `timezone`
    - `optimal_send_hour`
    - `best_open_rate`
    - `hour_stats`
    - `analyzed_at`
- **Key expressions or variables used:**  
  Uses `$input.all()`
- **Input and output connections:**  
  - Input: `Get Performance Data`
  - Output: `Update Send Time Rules`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - Strong mismatch with prior SQL output (`hour`, `open_count`) means this logic will not produce correct results as written
  - If no valid metrics are generated, no items are returned
- **Sub-workflow reference:**  
  None

#### 28. Update Send Time Rules
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Upserts the best send hour per timezone.
- **Configuration choices:**  
  - Table: `send_time_rules`
  - Matching column: `timezone`
  - Writes:
    - `timezone = {{$json.timezone}}`
    - `last_updated = {{$now.toISO()}}`
    - `best_send_hour = {{$json.best_send_hour}}`
- **Key expressions or variables used:**  
  - `{{$json.timezone}}`
  - `{{$json.best_send_hour}}`
- **Input and output connections:**  
  - Input: `Calculate Best Send Times`
  - Output: none
- **Version-specific requirements:**  
  Postgres version `2.6`.
- **Edge cases or potential failure types:**  
  - Upstream code outputs `optimal_send_hour`, not `best_send_hour`
  - Upsert may silently write null/empty values
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Campaign Upload Webhook | Webhook | Receives campaign data and CSV upload |  | Workflow Configuration | ## Campaign Setup<br>Receives campaign data and CSV leads. Configures limits, delays, and validation settings. |
| Workflow Configuration | Set | Injects central config values for intake processing | Campaign Upload Webhook | Extract CSV Leads | ## Campaign Setup<br>Receives campaign data and CSV leads. Configures limits, delays, and validation settings. |
| Extract CSV Leads | Extract From File | Parses uploaded CSV into structured records | Workflow Configuration | Split Leads | ## Lead Extraction<br>Parses uploaded CSV and splits leads into individual records for processing. |
| Split Leads | Split Out | Splits parsed lead array into one item per lead | Extract CSV Leads | Validate Email & Extract Domain | ## Lead Extraction<br>Parses uploaded CSV and splits leads into individual records for processing. |
| Validate Email & Extract Domain | Code | Validates email syntax and extracts domain | Split Leads | Is Email Valid? | ## Lead Extraction<br>Parses uploaded CSV and splits leads into individual records for processing.<br>## Validation Routing<br>Valid leads continue to processing. Invalid ones are stored separately. |
| Is Email Valid? | If | Routes valid and invalid emails | Validate Email & Extract Domain | Check Domain MX Records; Store Invalid Lead | ## Validation Routing<br>Valid leads continue to processing. Invalid ones are stored separately. |
| Check Domain MX Records | HTTP Request | Checks MX records for lead domain | Is Email Valid? | Enrich Lead Data | ## Validation Routing<br>Valid leads continue to processing. Invalid ones are stored separately. |
| Store Invalid Lead | Postgres | Stores invalid leads in database | Is Email Valid? |  | ## Validation Routing<br>Valid leads continue to processing. Invalid ones are stored separately. |
| Enrich Lead Data | Code | Adds timezone, domain type, and engagement score | Check Domain MX Records | Store Valid Lead | ## Lead Enrichment<br>Verifies domain via MX records, assigns timezone, and calculates engagement score before storing. |
| Store Valid Lead | Postgres | Stores accepted leads in database | Enrich Lead Data |  | ## Lead Enrichment<br>Verifies domain via MX records, assigns timezone, and calculates engagement score before storing. |
| Campaign Scheduler | Schedule Trigger | Periodically starts campaign execution |  | Get Pending Campaigns | ## Campaign Execution<br>Fetches active campaigns and prepares them for email sending. |
| Get Pending Campaigns | Postgres | Fetches active campaigns ready to send | Campaign Scheduler | Split Campaigns | ## Campaign Execution<br>Fetches active campaigns and prepares them for email sending. |
| Split Campaigns | Split Out | Splits campaign result set into one item per campaign | Get Pending Campaigns | Get Campaign Leads | ## Campaign Execution<br>Fetches active campaigns and prepares them for email sending. |
| Get Campaign Leads | Postgres | Fetches top pending leads for a campaign | Split Campaigns | Split Campaign Leads | ## Lead Selection<br>Pulls top leads sorted by engagement score and readiness. |
| Split Campaign Leads | Split Out | Splits campaign lead result set into one item per lead | Get Campaign Leads | Calculate Send Time & Select Inbox | ## Lead Selection<br>Pulls top leads sorted by engagement score and readiness. |
| Calculate Send Time & Select Inbox | Code | Computes target send time and picks an inbox under limits | Split Campaign Leads | Wait Until Send Time | ## Send Optimization<br>Calculates best send time per lead and selects inbox using rotation and limits. |
| Wait Until Send Time | Wait | Pauses execution until lead-specific send time | Calculate Send Time & Select Inbox | Send Email | ## Email Delivery<br>Waits until optimal time and sends emails using selected inbox. |
| Send Email | Gmail | Sends outbound email | Wait Until Send Time | Log Send Event | ## Email Delivery<br>Waits until optimal time and sends emails using selected inbox. |
| Log Send Event | Postgres | Records sent email event | Send Email | Update Inbox Stats | ## Send Tracking<br>Logs sent emails and updates inbox usage statistics. |
| Update Inbox Stats | Postgres | Increments inbox usage counters | Log Send Event |  | ## Send Tracking<br>Logs sent emails and updates inbox usage statistics. |
| Email Event Webhook | Webhook | Receives email activity events |  | Parse Event Data | ## Event Intake<br>Receives email events like opens, clicks, replies, and bounces. |
| Parse Event Data | Code | Normalizes tracking event payloads | Email Event Webhook | Log Email Event | ## Event Processing<br>Parses event data and logs it for tracking and analytics. |
| Log Email Event | Postgres | Stores normalized email events | Parse Event Data | Update Performance Stats | ## Event Processing<br>Parses event data and logs it for tracking and analytics.<br>## Performance Update<br>Updates campaign performance metrics based on user interactions. |
| Update Performance Stats | Postgres | Updates campaign aggregate engagement stats | Log Email Event |  | ## Performance Update<br>Updates campaign performance metrics based on user interactions. |
| Analytics Scheduler | Schedule Trigger | Periodically starts analytics recalculation |  | Get Performance Data | ## Analytics Trigger<br>Runs periodically to analyze campaign performance. |
| Get Performance Data | Postgres | Queries historical open-performance data | Analytics Scheduler | Calculate Best Send Times | ## Performance Analysis<br>Analyzes historical data to identify best-performing send times. |
| Calculate Best Send Times | Code | Derives optimal send hour by timezone | Get Performance Data | Update Send Time Rules | ## Performance Analysis<br>Analyzes historical data to identify best-performing send times.<br>## Continuous Optimization<br>Updates send-time rules to improve future campaign performance. |
| Update Send Time Rules | Postgres | Upserts timezone send-time rules | Calculate Best Send Times |  | ## Continuous Optimization<br>Updates send-time rules to improve future campaign performance. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n.

## A. Prepare dependencies first

1. **Create Postgres credentials**
   - In n8n, add a Postgres credential with host, port, database, username, password, and SSL settings as required.

2. **Create Gmail credentials**
   - Add a Gmail OAuth2 credential.
   - Ensure the Google account used is authorized to send mail.
   - Note: this workflow conceptually rotates inboxes, but the native Gmail node credential is static per node unless you design credential switching another way.

3. **Prepare database tables**
   At minimum, create tables similar to:
   - `campaigns`
   - `leads`
   - `invalid_leads`
   - `email_events`
   - `campaign_stats`
   - `send_time_rules`
   - `inbox_stats`

4. **Decide CSV upload format**
   - The webhook request should include campaign metadata and a CSV file in binary form.
   - Ensure CSV headers include at least `email`.
   - If using `campaign_id`, include it either in each row or merge it before storage.

---

## B. Build the lead ingestion branch

1. **Add a Webhook node**
   - Name: `Campaign Upload Webhook`
   - Path: `campaign-upload`
   - Method: `POST`
   - Response Mode: `Last Node`

2. **Add a Set node**
   - Name: `Workflow Configuration`
   - Connect from `Campaign Upload Webhook`
   - Add fields:
     - `maxEmailsPerInboxPerHour` = `50`
     - `maxEmailsPerInboxPerDay` = `500`
     - `minDelayBetweenEmailsMinutes` = `2`
     - `mxRecordCheckApiUrl` = `https://dns.google/resolve`
     - `defaultTimezone` = `UTC`
   - Enable keeping incoming fields

3. **Add an Extract From File node**
   - Name: `Extract CSV Leads`
   - Connect from `Workflow Configuration`
   - Configure it to parse the uploaded CSV binary field used by your webhook request

4. **Add a Split Out node**
   - Name: `Split Leads`
   - Connect from `Extract CSV Leads`
   - Field to split: `data`

5. **Add a Code node**
   - Name: `Validate Email & Extract Domain`
   - Connect from `Split Leads`
   - Mode: `Run Once for Each Item`
   - Paste logic that:
     - reads `email`
     - validates syntax with regex
     - extracts domain after `@`
     - returns original row plus `isValid`, `domain`, `validatedAt`

6. **Add an If node**
   - Name: `Is Email Valid?`
   - Connect from `Validate Email & Extract Domain`
   - Condition: boolean true on `isValid`

7. **Add a Postgres node for invalid leads**
   - Name: `Store Invalid Lead`
   - Connect from the false output of `Is Email Valid?`
   - Operation: insert row into `public.invalid_leads`
   - Map:
     - `email`
     - `reason` = `invalid_syntax_or_domain`
     - `campaign_id`

8. **Add an HTTP Request node**
   - Name: `Check Domain MX Records`
   - Connect from the true output of `Is Email Valid?`
   - Method: `GET`
   - URL:
     `={{ $('Workflow Configuration').first().json.mxRecordCheckApiUrl }}?name={{ $json.domain }}&type=MX`
   - Response format: JSON
   - Recommended improvement: enable options to preserve original input data or merge response with input

9. **Add a Code node**
   - Name: `Enrich Lead Data`
   - Connect from `Check Domain MX Records`
   - Mode: `Run Once for Each Item`
   - Implement logic that:
     - derives `domainType`
     - estimates `timezone` from TLD
     - computes `engagementScore`
     - returns original data plus enrichment fields

10. **Add a Postgres node**
    - Name: `Store Valid Lead`
    - Connect from `Enrich Lead Data`
    - Insert into `public.leads`
    - Recommended mapped columns:
      - `email`
      - `domain`
      - `status` = `pending`
      - `timezone`
      - `campaign_id`
      - `domain_type`
      - `engagement_score`
    - Important: align field names with the Code node output. Either:
      - output snake_case in the Code node, or
      - map `domain_type = {{$json.domainType}}` and `engagement_score = {{$json.engagementScore}}`

---

## C. Build the campaign execution branch

11. **Add a Schedule Trigger node**
    - Name: `Campaign Scheduler`
    - Configure to run every minute, or another safe interval

12. **Add a Postgres node**
    - Name: `Get Pending Campaigns`
    - Connect from `Campaign Scheduler`
    - Operation: execute query
    - Query:
      `SELECT * FROM campaigns WHERE status = 'active' AND scheduled_time <= NOW()`

13. **Add a Split Out node**
    - Name: `Split Campaigns`
    - Connect from `Get Pending Campaigns`
    - Field to split: `data`

14. **Add a Postgres node**
    - Name: `Get Campaign Leads`
    - Connect from `Split Campaigns`
    - Operation: execute query
    - Query similar to:
      `SELECT l.*, st.best_send_hour FROM leads l LEFT JOIN send_time_rules st ON l.timezone = st.timezone WHERE l.campaign_id = {{ $json.id }} AND l.status = 'pending' ORDER BY l.engagement_score DESC LIMIT 100`
    - Recommended improvement:
      - include `subject`, `emailBody`, and `lead_id` if needed
      - include inbox assignment data, or fetch inboxes in a dedicated node

15. **Add a Split Out node**
    - Name: `Split Campaign Leads`
    - Connect from `Get Campaign Leads`
    - Field to split: `data`

16. **Add a Code node**
    - Name: `Calculate Send Time & Select Inbox`
    - Connect from `Split Campaign Leads`
    - Mode: `Run Once for Each Item`
    - Implement:
      - timezone-aware send time calculation
      - fallback to next occurrence of `best_send_hour`
      - inbox selection from an `available_inboxes` array
    - Strong recommendation:
      - add a prior Postgres node to fetch eligible inboxes and merge them into each lead item
      - avoid relying on `require('luxon')` unless your n8n environment supports it

17. **Add a Wait node**
    - Name: `Wait Until Send Time`
    - Connect from `Calculate Send Time & Select Inbox`
    - Resume: `Specific Time`
    - Date/Time: `={{ $json.sendTime }}`

18. **Add a Gmail node**
    - Name: `Send Email`
    - Connect from `Wait Until Send Time`
    - Credential: Gmail OAuth2
    - To: `={{ $json.email }}`
    - Subject: `={{ $json.subject }}`
    - Message: `={{ $json.emailBody }}`
    - Optional sender name: use a string such as `={{ $json.selectedInbox.email }}`
    - Important limitation:
      - This does not rotate actual sender accounts unless the node/credential architecture is redesigned
      - For true rotation, use separate branches, separate credentials, SMTP, or a sub-workflow pattern

---

## D. Build send logging and inbox counter updates

19. **Add a Postgres node**
    - Name: `Log Send Event`
    - Connect from `Send Email`
    - Insert into `public.email_events`
    - Map recommended fields:
      - `lead_id`
      - `campaign_id`
      - `event_type` = `sent`
      - `sent_at` or `event_timestamp` = current time
      - `inbox_used = {{$json.selectedInbox.email}}`

20. **Add a Postgres node**
    - Name: `Update Inbox Stats`
    - Connect from `Log Send Event`
    - Execute query to increment:
      - `emails_sent_today`
      - `emails_sent_this_hour`
      - `last_send_time`
    - Recommended query form:
      `UPDATE inbox_stats SET emails_sent_today = emails_sent_today + 1, emails_sent_this_hour = emails_sent_this_hour + 1, last_send_time = NOW() WHERE inbox_email = '{{ $json.selectedInbox.email }}'`
    - Prefer parameterized queries if available

---

## E. Build the event intake branch

21. **Add a Webhook node**
    - Name: `Email Event Webhook`
    - Path: `email-events`
    - Method: `POST`

22. **Add a Code node**
    - Name: `Parse Event Data`
    - Connect from `Email Event Webhook`
    - Mode: `Run Once for Each Item`
    - Normalize payload into a consistent structure
    - Recommended output keys:
      - `event_type`
      - `email_identifier`
      - `event_timestamp`
      - `recipient_email`
      - `campaign_id`
      - `lead_id`
      - `event_details`
      - `raw_payload`
    - This avoids the snake_case/camelCase mismatch present in the provided workflow

23. **Add a Postgres node**
    - Name: `Log Email Event`
    - Connect from `Parse Event Data`
    - Insert into `public.email_events`
    - Map fields using the parser’s exact output names

24. **Add a Postgres node**
    - Name: `Update Performance Stats`
    - Connect from `Log Email Event`
    - Execute query to increment:
      - opens when `event_type = 'open'`
      - clicks when `event_type = 'click'`
      - replies when `event_type = 'reply'`
    - Recommended to use clean normalized fields and parameterized values

---

## F. Build the analytics optimization branch

25. **Add a Schedule Trigger node**
    - Name: `Analytics Scheduler`
    - Configure a periodic interval, for example every day or every few hours

26. **Add a Postgres node**
    - Name: `Get Performance Data`
    - Connect from `Analytics Scheduler`
    - Query historical engagement data by timezone and hour
    - Recommended query should return fields that match your code logic, for example:
      - `timezone`
      - `send_hour`
      - `total_sent`
      - `total_opened`

27. **Add a Code node**
    - Name: `Calculate Best Send Times`
    - Connect from `Get Performance Data`
    - Group rows by timezone and send hour
    - Compute weighted open rate
    - Output:
      - `timezone`
      - `best_send_hour`
      - `best_open_rate`
      - `analyzed_at`

28. **Add a Postgres node**
    - Name: `Update Send Time Rules`
    - Connect from `Calculate Best Send Times`
    - Operation: upsert into `public.send_time_rules`
    - Matching key: `timezone`
    - Fields:
      - `timezone`
      - `best_send_hour`
      - `last_updated`

---

## G. Add sticky notes or operational notes

29. Add sticky notes for:
   - Campaign Setup
   - Lead Extraction
   - Validation Routing
   - Lead Enrichment
   - Campaign Execution
   - Lead Selection
   - Send Optimization
   - Email Delivery
   - Send Tracking
   - Event Intake
   - Event Processing
   - Performance Update
   - Analytics Trigger
   - Performance Analysis
   - Continuous Optimization

---

## H. Important fixes required for a working implementation

30. **Fix field name mismatches**
   - `domainType` vs `domain_type`
   - `engagementScore` vs `engagement_score`
   - `eventType` vs `event_type`
   - `campaignId` vs `campaign_id`
   - `leadId` vs `lead_id`
   - `timestamp` vs `event_timestamp`
   - `optimal_send_hour` vs `best_send_hour`

31. **Add inbox source data**
   - The workflow expects `available_inboxes`, but no node creates it.
   - Add a Postgres query or another source before `Calculate Send Time & Select Inbox`.

32. **Provide email content fields**
   - `Send Email` uses `subject` and `emailBody`, but they are not created anywhere in the workflow.
   - Fetch them from the campaign record or generate them before sending.

33. **Correct Gmail sender handling**
   - `senderName` should be text, not the full `selectedInbox` object.
   - Gmail node credentials do not automatically switch based on item data.

34. **Review HTTP Request data preservation**
   - Ensure MX lookup does not discard original lead fields.

35. **Correct analytics SQL/code alignment**
   - The analytics SQL and code currently expect different field sets.

36. **Secure SQL expressions**
   - Prefer parameterized queries over string interpolation wherever possible.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end email campaigns with built-in deliverability protection and smart optimization. It ingests campaign data and CSV leads, validates emails, checks domains via MX records, and enriches leads with timezone and engagement scores. Campaigns are executed using a scheduler that selects top leads and calculates optimal send times based on timezone and past performance. Emails are sent through rotating inboxes with strict limits, while all events are tracked to continuously improve performance. | General workflow concept |
| Setup guidance included in the workflow: connect webhook for campaign and CSV upload; configure send limits, delays, and MX API; set up Postgres with required tables; connect Gmail or SMTP for sending; configure event webhook for tracking; enable campaign and analytics schedulers; test with sample campaign before activating. | Operational setup note |

## Additional implementation observations

- The design is sound conceptually, but the provided workflow JSON is **partially incomplete/inconsistent** and will require correction before production use.
- The biggest functional gaps are:
  - no inbox retrieval step
  - no campaign content retrieval for subject/body
  - several field naming mismatches between Code nodes and Postgres nodes
  - analytics query/output mismatch
  - Gmail sender rotation not implemented at credential level
- For true reputation-safe bulk sending, consider:
  - adding bounce suppression logic
  - updating lead status after send
  - adding inbox cooldown periods
  - implementing per-domain pacing
  - securing event webhooks with signature validation